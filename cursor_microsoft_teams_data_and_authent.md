# Microsoft Teams Integration Documentation

## Overview

This integration enables users to connect their Microsoft Teams accounts, sync their chats, messages, teams, channels, and channel posts, and archive them to S3 storage.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Data Models                                    │
├────────────────────┬────────────────────┬───────────────────────────────┤
│  TeamsChat         │  TeamsCommunity    │  TeamsCommunityChannelPost    │
│  TeamsMessage      │  TeamsCommunityChannel                             │
└────────────────────┴────────────────────┴───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────────────────┐
│                           Services                                       │
├────────────────────┬────────────────────┬───────────────────────────────┤
│  TeamsOAuthService │  TeamsChatService  │  TeamsSyncService             │
│  (Authentication)  │  (1:1/Group Chats) │  (Teams/Channels/Posts)       │
└────────────────────┴────────────────────┴───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────────────────┐
│                      Views (HTTP Endpoints)                              │
│  login() → authorized() → get_texts() → sync & archive                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Authentication Flow

### OAuth 2.0 Authorization Code Flow

The system uses Microsoft's OAuth 2.0 authorization code flow with the following endpoints:

| Endpoint | URL |
|----------|-----|
| Authorize | `https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/authorize` |
| Token | `https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token` |
| Graph API | `https://graph.microsoft.com/v1.0` |

### Step 1: Initiate Login

**Endpoint:** `GET /text/microsoft_teams/login/`

```python
# views/microsoft_teams.py - login()
auth_url = (
    f'{AUTHORIZE_URL}?client_id={MICROSOFT_CLIENT_ID}'
    f'&redirect_uri={redirect_uri}'
    f'&scope={scope}'
    f'&response_type=code'
    f'&response_mode=query'
    f'&prompt=consent'
)
auth_url += f'&state={request.user.id}'
```

**Required Scopes** (from `my_settings.py`):
- `openid`
- `profile`
- `offline_access` (for refresh tokens)
- `User.Read`
- `Chat.Read`
- `Team.ReadBasic.All`
- `Channel.ReadBasic.All`
- `ChannelMessage.Read.All`

### Step 2: Handle Callback

**Endpoint:** `GET /text/microsoft_teams/callback/` → `authorized()`

```python
# 1. Extract authorization code from query params
code = request.GET['code']
request_user_id = request.GET['state']

# 2. Exchange code for tokens
short_token = get_short_live_access_token(code, redirect_uri)
```

### Step 3: Exchange Code for Tokens

```python
def get_short_live_access_token(code, redirect_uri):
    data = {
        'client_id': MICROSOFT_CLIENT_ID,
        'client_secret': MICROSOFT_CLIENT_SECRET,
        'code': code,
        'grant_type': 'authorization_code',
        'redirect_uri': redirect_uri,
        'scope': ' '.join(MICROSOFT_SCOPES),
    }
    resp = requests.post(TOKEN_URL, data=data)
    return resp.json()
```

**Token Response:**
```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLC...",
    "refresh_token": "0.ARgA...",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```

### Step 4: Fetch User Profile & Save Account

```python
# Fetch profile from Graph API
resp = requests.get(f'{MICROSOFT_GRAPH_URL}/me', headers=headers)
extra_data = resp.json()

# Create/Update Account record
account = Account.objects.create(
    provider='Microsoft Teams',
    uid=extra_data.get('id'),
    name=extra_data.get('displayName'),
    username=extra_data.get('mail') or extra_data.get('userPrincipalName'),
    token=access_token,
    token_secret=refresh_token,  # Stored for token refresh
    expires_at=expires_at,
    user=user
)
```

### Token Refresh Flow

When the access token expires, the system automatically refreshes it:

```python
# TeamsOAuthService.refresh_access_token()
def refresh_access_token(cls, refresh_token: str) -> dict:
    data = {
        'client_id': MICROSOFT_CLIENT_ID,
        'client_secret': MICROSOFT_CLIENT_SECRET,
        'refresh_token': refresh_token,
        'grant_type': 'refresh_token',
        'scope': final_scopes,
    }
    response = cls._post_form(cls.TOKEN_URL, data)
    return response.json()
```

**Auto-refresh in API calls:**

```python
# TeamsChatService._request_with_refresh()
if resp.status_code == 401:
    # Token expired - refresh and retry
    new_token_data = TeamsOAuthService.refresh_access_token(self.account.token_secret)
    self.account.token = new_token_data["access_token"]
    self.account.save()
    # Retry request with new token
```

---

## 2. Fetching Teams Chats & Messages

### Data Models

**TeamsChat:**
```python
class TeamsChat(models.Model):
    CHAT_TYPES = [
        ('oneOnOne', 'One on One'),
        ('group', 'Group'),
        ('meeting', 'Meeting'),
    ]
    
    user = models.ForeignKey(AUTH_USER_MODEL)
    account = models.ForeignKey(Account)
    chat_id = models.CharField(max_length=255)  # Graph chat.id
    topic = models.CharField(max_length=255)
    chat_type = models.CharField(choices=CHAT_TYPES)
```

**TeamsMessage:**
```python
class TeamsMessage(models.Model):
    chat = models.ForeignKey(TeamsChat)
    message_id = models.CharField(unique=True)
    sender_id = models.CharField()
    sender_name = models.CharField()
    sender_photo = models.TextField()  # data:image/jpeg;base64,...
    body_html = models.TextField()
    created_date = models.DateTimeField()
    is_from_me = models.BooleanField()
```

### Fetch Chats Flow

**Service:** `TeamsChatService`

```python
# 1. Fetch all chats via Graph API
def fetch_chats(self, limit: int = 50) -> List[dict]:
    url = f"{self.GRAPH_URL}/me/chats"
    params = {
        "$top": min(limit, 50),
        "$expand": "members",
    }
    chats = self._get_paginated(url, params)
    # Filter valid types
    valid_types = {"oneOnOne", "group", "meeting"}
    return [c for c in chats if c.get("chatType") in valid_types]
```

**Graph API Endpoint:** `GET /me/chats?$expand=members&$top=50`

**Response:**
```json
{
    "value": [
        {
            "id": "19:meeting_abc123...",
            "topic": "Project Discussion",
            "chatType": "group",
            "createdDateTime": "2024-01-15T10:00:00Z",
            "members": [...]
        }
    ],
    "@odata.nextLink": "..."
}
```

### Fetch Messages Flow

```python
def fetch_messages(self, chat_id: str, limit: int = 50, since: Optional[datetime] = None) -> List[dict]:
    url = f"{self.GRAPH_URL}/chats/{chat_id}/messages"
    params = {"$top": min(limit, 50)}
    messages = self._get_paginated(url, params)
    return messages
```

**Graph API Endpoint:** `GET /chats/{chat_id}/messages?$top=50`

### Sync All Chats & Messages

```python
def sync_all(self, messages_per_chat: int = 50) -> Dict[str, Any]:
    # 1. Sync all chats first
    saved_chats = self.sync_chats()
    
    # 2. For each chat, sync messages
    for chat in saved_chats:
        messages = self.sync_messages(chat.chat_id, messages_per_chat)
    
    return results
```

### Sender Photo Retrieval

```python
def _get_user_photo(self, user_id: str) -> Optional[str]:
    url = f"{self.GRAPH_URL}/users/{user_id}/photo/$value"
    resp = session.get(url, headers=self.headers)
    if resp.status_code == 200:
        encoded = base64.b64encode(resp.content).decode("utf-8")
        return f"data:{content_type};base64,{encoded}"
```

---

## 3. Fetching Teams, Channels & Channel Posts

### Data Models

**TeamsCommunity (Team):**
```python
class TeamsCommunity(models.Model):
    team_id = models.CharField(unique=True)  # Graph team.id
    display_name = models.CharField()
    description = models.TextField()
    visibility = models.CharField(choices=[('private', 'Private'), ('public', 'Public')])
    is_archived = models.BooleanField()
```

**TeamsCommunityChannel:**
```python
class TeamsCommunityChannel(models.Model):
    CHANNEL_TYPES = [
        ('standard', 'Standard'),
        ('private', 'Private'),
        ('shared', 'Shared'),
    ]
    
    community = models.ForeignKey(TeamsCommunity)
    channel_id = models.CharField(unique=True)
    display_name = models.CharField()
    channel_type = models.CharField(choices=CHANNEL_TYPES)
```

**TeamsCommunityChannelPost:**
```python
class TeamsCommunityChannelPost(models.Model):
    community = models.ForeignKey(TeamsCommunity)
    channel = models.ForeignKey(TeamsCommunityChannel)
    post_id = models.CharField(unique=True)
    parent_post = models.ForeignKey('self', null=True)  # For replies
    subject = models.CharField()
    body_html = models.TextField()
    sender_id = models.CharField()
    sender_name = models.CharField()
    posted_at = models.DateTimeField()
    is_root_post = models.BooleanField()
```

### Fetch Teams (Communities) Flow

**Service:** `TeamsSyncService`

```python
def sync_communities(self, token: str) -> List[TeamsCommunity]:
    url = f"{TeamsOAuthService.GRAPH_URL}/me/joinedTeams"
    
    while next_url:
        data = self._fetch_graph(next_url, token)
        for item in data.get("value", []):
            TeamsCommunity.objects.update_or_create(
                team_id=item.get("id"),
                defaults={
                    "display_name": item.get("displayName"),
                    "description": item.get("description"),
                    "visibility": item.get("visibility"),
                    "is_archived": item.get("isArchived"),
                }
            )
        next_url = data.get("@odata.nextLink")
```

**Graph API Endpoint:** `GET /me/joinedTeams`

### Fetch Channels Flow

```python
def sync_community_channels(self, community: TeamsCommunity, token: str) -> List[TeamsCommunityChannel]:
    team_id = community.team_id
    url = f"{TeamsOAuthService.GRAPH_URL}/teams/{team_id}/channels"
    
    while next_url:
        data = self._fetch_graph(next_url, token)
        for item in data.get("value", []):
            TeamsCommunityChannel.objects.update_or_create(
                channel_id=item.get("id"),
                defaults={
                    "community": community,
                    "display_name": item.get("displayName"),
                    "channel_type": item.get("membershipType"),
                }
            )
```

**Graph API Endpoint:** `GET /teams/{team_id}/channels`

### Fetch Channel Posts (Messages) Flow

```python
def sync_channel_posts(self, community: TeamsCommunity, channel: TeamsCommunityChannel, token: str, limit: int = 50) -> int:
    team_id = community.team_id
    channel_id = channel.channel_id
    url = f"{TeamsOAuthService.GRAPH_URL}/teams/{team_id}/channels/{channel_id}/messages"
    
    while next_url and remaining > 0:
        data = self._fetch_graph(next_url, token, params={"$top": 50})
        for item in data.get("value", []):
            # Handle reply threading
            parent_post_id = item.get("replyToId")
            is_root = parent_post_id is None
            
            TeamsCommunityChannelPost.objects.update_or_create(
                post_id=item.get("id"),
                defaults={
                    "community": community,
                    "channel": channel,
                    "subject": item.get("subject"),
                    "body_html": item.get("body", {}).get("content"),
                    "sender_id": sender_id,
                    "sender_name": sender_name,
                    "posted_at": item.get("createdDateTime"),
                    "is_root_post": is_root,
                }
            )
```

**Graph API Endpoint:** `GET /teams/{team_id}/channels/{channel_id}/messages?$top=50`

### Complete Sync Flow

```python
# TeamsSyncService.sync_all()
def sync_all(self, sync_posts_per_channel: int = 50) -> Dict[str, int]:
    token = self._get_token()
    
    # 1. Sync all teams
    teams = self.sync_communities(token)
    
    # 2. For each team, sync channels
    for team in teams:
        channels = self.sync_community_channels(team, token)
        
        # 3. For each channel, sync posts
        for channel in channels:
            self.sync_channel_posts(team, channel, token, limit=sync_posts_per_channel)
```

---

## 4. Data Flow Diagram

```
User clicks "Connect Microsoft Teams"
           │
           ▼
┌─────────────────────────────────────┐
│  login() - Generate Auth URL        │
│  Redirect to Microsoft Login        │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  Microsoft OAuth Consent Screen     │
│  User grants permissions            │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  authorized() - Callback Handler    │
│  1. Extract auth code               │
│  2. Exchange for tokens             │
│  3. Fetch user profile (/me)        │
│  4. Save Account record             │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  get_texts() - Main Sync Entry      │
└─────────────────────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌────────────┐  ┌────────────────────┐
│ TeamsChatService │  │ TeamsSyncService   │
│ sync_all() │      │ sync_all()         │
└────────────┘      └────────────────────┘
     │                     │
     ▼                     ▼
┌────────────┐      ┌────────────────────┐
│ /me/chats  │      │ /me/joinedTeams    │
│ /chats/{id}│      │ /teams/{id}/channels│
│ /messages  │      │ /channels/{id}/messages│
└────────────┘      └────────────────────┘
     │                     │
     ▼                     ▼
┌────────────┐      ┌────────────────────┐
│ TeamsChat  │      │ TeamsCommunity     │
│TeamsMessage│      │TeamsCommunityChannel│
└────────────┘      │TeamsCommunityChannelPost│
                    └────────────────────┘
```

---

## 5. API Endpoints Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/text/microsoft_teams/login/` | GET/POST | Initiate OAuth login |
| `/text/microsoft_teams/callback/` | GET | OAuth callback handler |
| `/text/microsoft_teams/disconnect/` | POST | Disconnect account |

### Microsoft Graph API Endpoints Used

| Endpoint | Purpose |
|----------|---------|
| `GET /me` | Fetch user profile |
| `GET /me/photo/$value` | Fetch user profile photo |
| `GET /me/chats` | List user's chats |
| `GET /chats/{chat_id}/messages` | Get messages in a chat |
| `GET /me/joinedTeams` | List teams user is member of |
| `GET /teams/{team_id}/channels` | List channels in a team |
| `GET /teams/{team_id}/channels/{channel_id}/messages` | Get channel messages |
| `GET /users/{user_id}/photo/$value` | Fetch any user's photo |

---

## 6. Error Handling

### Token Expiration
- Access tokens expire after ~1 hour
- System auto-refreshes using `refresh_token` stored in `Account.token_secret`
- If refresh fails, user is notified to reconnect

### 401 Unauthorized
```python
if resp.status_code == 401:
    # Attempt token refresh
    new_token_data = TeamsOAuthService.refresh_access_token(self.account.token_secret)
    # Retry request
```

### 403 Forbidden
```python
if resp.status_code == 403:
    # User may not have Teams license
    logger.warning("User may not have valid Teams license")
    return {"value": []}  # Return empty, don't fail
```

### Missing Refresh Token
```python
if not refresh_token:
    Account.objects.filter(id=account.id).update(status="missing_refresh_token")
    # Notify user to reconnect
```

---

## 7. Archiving

Data is archived to AWS S3 in CSV format:

```
{ENVIRONMENT}/archive_data/{user_id}/Microsoft Teams/{account_id}/{year}/
├── {username}--chat.csv
├── {username}--message.csv
├── {username}--community.csv
├── {username}--channel.csv
└── {username}--post.csv
```

