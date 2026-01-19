# BlueSky API Documentation

## Overview

This document describes all BlueSky-related API endpoints in the SharpArchive Django API. The integration supports OAuth 2.0 with DPoP (Demonstrating Proof-of-Possession) for secure authentication, fetching posts, comments, chats, and messages from BlueSky accounts.

---

## Table of Contents

1. [Authentication Endpoints](#1-authentication-endpoints)
   - [1.1 Initiate OAuth Login](#11-initiate-oauth-login)
   - [1.2 OAuth Callback (Authorized)](#12-oauth-callback-authorized)
   - [1.3 Connect Account (API)](#13-connect-account-api)
   - [1.4 Disconnect Account](#14-disconnect-account)
2. [Internal Service Methods](#2-internal-service-methods)
   - [2.1 Get Posts](#21-get-posts)
   - [2.2 Get Comments](#22-get-comments)
   - [2.3 Get Chats](#23-get-chats)
   - [2.4 Get Messages](#24-get-messages)
   - [2.5 Archive](#25-archive)
3. [External BlueSky API Endpoints Used](#3-external-bluesky-api-endpoints-used)
4. [Error Codes](#4-error-codes)
5. [Data Models](#5-data-models)

---

## 1. Authentication Endpoints

### 1.1 Initiate OAuth Login

Initiates the BlueSky OAuth 2.0 authorization flow with PKCE.

**Endpoint:** `GET /social/bluesky/login/`

**Authentication:** Required (Django login)

**Request:**
```http
GET /social/bluesky/login/ HTTP/1.1
Host: api.sharparchive.com
Cookie: sessionid=<session_token>
```

**Response:**

| Status | Description |
|--------|-------------|
| 302 | Redirect to BlueSky authorization URL |
| 400 | Error initializing authorization |

**Success Response:**
```json
{
    "status": 302,
    "Location": "https://bsky.social/oauth/authorize?client_id=<client_id>&request_uri=<request_uri>&state=<state>"
}
```

**Error Response:**
```json
{
    "status": 400,
    "message": "Error description"
}
```

**OAuth Scopes Requested:**
- `atproto` - Base AT Protocol scope
- `transition:generic` - Base authentication
- `transition:chat.bsky` - Chat access
- `rpc:chat.bsky.convo.listConvos?aud=did:web:api.bsky.chat`
- `rpc:chat.bsky.convo.getMessages?aud=did:web:api.bsky.chat`
- `rpc:chat.bsky.convo.sendMessage?aud=did:web:api.bsky.chat`

---

### 1.2 OAuth Callback (Authorized)

Handles the OAuth callback after user authorization on BlueSky.

**Endpoint:** `GET /social/bluesky/authorized/`

**Authentication:** Required (Django login)

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | string | Yes | Authorization code from BlueSky |
| `state` | string | Yes | State parameter for CSRF protection |
| `error` | string | No | Error code if authorization failed |
| `error_description` | string | No | Error description |

**Request:**
```http
GET /social/bluesky/authorized/?code=abc123&state=user_id:uuid HTTP/1.1
Host: api.sharparchive.com
Cookie: sessionid=<session_token>
```

**Response:**

| Status | Description |
|--------|-------------|
| 200 | Success - Returns JavaScript to close popup |
| 400 | Missing authorization code or general error |
| 401 | Authentication denied or failed |

**Success Response:**
Returns HTML/JavaScript to close the popup window:
```html
<script type='text/javascript'>
    opener.dismissPopup(window, 'Bluesky');
</script>
```

**Error Response:**
```json
{
    "status": 401,
    "message": "Authentication Denied"
}
```

**Side Effects:**
- Creates/updates `Account` record
- Creates/updates `AccountProfile` record
- Stores DPoP keys in `AccountMeta`
- Creates `AccountHistory` entry (on new connection)
- Fetches initial posts and chats
- Starts Jetstream firehose listener (background thread)
- Sends email notification on new connection

---

### 1.3 Connect Account (API)

API endpoint for connecting BlueSky account (used by mobile/external clients).

**Endpoint:** `POST /api/social/bluesky/`

**Authentication:** JWT Bearer token or Guest token

**Request Headers:**
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | Yes | OAuth authorization code |
| `state` | string | No | Guest token for delegated auth |

**Request:**
```json
{
    "code": "authorization_code_from_bluesky",
    "state": "guest_token_if_applicable"
}
```

**Response:**

| Status | Description |
|--------|-------------|
| 200 | Success |
| 400 | General error |
| 401 | Authentication denied |
| 402 | Authentication failed |
| 403 | Invalid guest token |
| 404 | Expired guest token |
| 405 | User permission denied |
| 406 | No authentication credentials |
| 407 | Token exchange failed |
| 408 | Session fetch failed |

**Success Response:**
```json
{
    "status": 200,
    "success": true,
    "message": "Feed Connected"
}
```

**Possible Messages:**
- `"Feed Connected"` - New account connected
- `"Feed Updated"` - Existing account refreshed
- `"Demo Version"` - Demo environment response

**Error Response:**
```json
{
    "status": 407,
    "message": "Token exchange failed: invalid_grant"
}
```

---

### 1.4 Disconnect Account

Disconnects and revokes BlueSky account access.

**Endpoint:** `POST /social/bluesky/disconnect/`

**Authentication:** Required (Django login)

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `account_id` | integer | Yes | ID of the account to disconnect |

**Request:**
```http
POST /social/bluesky/disconnect/ HTTP/1.1
Host: api.sharparchive.com
Content-Type: application/x-www-form-urlencoded
Cookie: sessionid=<session_token>

account_id=123
```

**Response:**

| Status | Description |
|--------|-------------|
| 200 | Success |
| 400 | Missing account_id |
| 404 | Account not found |
| 500 | Internal error |

**Success Response:**
```json
{
    "status": 200,
    "message": "Account disconnected"
}
```

**Error Response:**
```json
{
    "status": 404,
    "message": "Account not found"
}
```

**Side Effects:**
- Revokes refresh and access tokens at BlueSky
- Clears stored tokens in database
- Updates `Account.date_closed`
- Updates `Account.status` to `"disconnected"`
- Deletes DPoP keys from `AccountMeta`
- Creates `AccountHistory` entry
- Clears session data

---

## 2. Internal Service Methods

These are internal methods called by the views. They are not directly exposed as API endpoints but document the data flow.

### 2.1 Get Posts

Fetches posts/feed from a BlueSky account.

**Method:** `BlueSkyService.get_posts(account, access_token, since_id, is_archive)`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account` | Account | Account model instance |
| `access_token` | string | OAuth access token |
| `since_id` | integer | Fetch posts since this ID (default: 1) |
| `is_archive` | boolean | True for full archive, False for incremental |

**BlueSky API Called:**
```
GET {pds_base}/xrpc/app.bsky.feed.getAuthorFeed
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `actor` | string | User DID or handle |
| `limit` | integer | Number of posts per page (max 100) |
| `cursor` | string | Pagination cursor |

**Response Data Structure (per post):**
```json
{
    "post": {
        "uri": "at://did:plc:xxx/app.bsky.feed.post/xxx",
        "cid": "bafyreiabc123...",
        "author": {
            "did": "did:plc:xxx",
            "handle": "user.bsky.social",
            "displayName": "Display Name",
            "avatar": "https://cdn.bsky.app/..."
        },
        "record": {
            "$type": "app.bsky.feed.post",
            "text": "Post content",
            "createdAt": "2024-01-15T12:00:00.000Z",
            "embed": {},
            "facets": [],
            "reply": {}
        },
        "embed": {
            "$type": "app.bsky.embed.images#view",
            "images": []
        },
        "replyCount": 5,
        "repostCount": 10,
        "likeCount": 25,
        "indexedAt": "2024-01-15T12:00:01.000Z"
    },
    "reply": {},
    "reason": {}
}
```

**Stored in Database:**
- `Bluesky` model with processed post data
- Media files downloaded to S3/local storage

---

### 2.2 Get Comments

Fetches comments/replies for a specific post thread.

**Method:** `BlueSkyService.get_comments(user_id, account_id, post_id, parent_uri, access_token, pds_base)`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | integer | User ID |
| `account_id` | integer | Account ID |
| `post_id` | integer | Internal post ID |
| `parent_uri` | string | AT Protocol URI of parent post |
| `access_token` | string | OAuth access token |
| `pds_base` | string | PDS base URL |

**BlueSky API Called:**
```
GET {pds_base}/xrpc/app.bsky.feed.getPostThread
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `uri` | string | Post AT URI |
| `depth` | integer | Thread depth (default: 100) |

**Response Data Structure:**
```json
{
    "thread": {
        "$type": "app.bsky.feed.defs#threadViewPost",
        "post": {
            "uri": "at://did:plc:xxx/app.bsky.feed.post/xxx",
            "cid": "bafyreiabc123...",
            "author": {},
            "record": {},
            "replyCount": 5,
            "repostCount": 10,
            "likeCount": 25
        },
        "replies": [
            {
                "$type": "app.bsky.feed.defs#threadViewPost",
                "post": {},
                "replies": []
            }
        ]
    }
}
```

**Stored in Database:**
- `BlueskyComment` model with nested reply structure

---

### 2.3 Get Chats

Fetches chat conversations from BlueSky.

**Method:** `BlueSkyService.get_chats(account, access_token, is_archive)`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account` | Account | Account model instance |
| `access_token` | string | OAuth access token |
| `is_archive` | boolean | True for full archive, False for incremental |

**BlueSky API Called (via PDS proxy):**
```
GET {pds_base}/xrpc/chat.bsky.convo.listConvos
```

**Required Headers:**
```http
Authorization: DPoP <access_token>
DPoP: <dpop_proof>
Atproto-Proxy: did:web:api.bsky.chat#bsky_chat
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Number of conversations per page (default: 100) |
| `cursor` | string | Pagination cursor |

**Response Data Structure:**
```json
{
    "convos": [
        {
            "id": "3abc123xyz",
            "rev": "3abc123xyz",
            "members": [
                {
                    "did": "did:plc:xxx",
                    "handle": "user.bsky.social",
                    "displayName": "Display Name",
                    "avatar": "https://cdn.bsky.app/..."
                }
            ],
            "muted": false,
            "unreadCount": 0,
            "messageCount": 15,
            "status": "allowed",
            "lastMessage": {
                "id": "msg123",
                "text": "Latest message",
                "sentAt": "2024-01-15T12:00:00.000Z",
                "sender": {
                    "did": "did:plc:xxx"
                }
            }
        }
    ],
    "cursor": "next_page_cursor"
}
```

**Stored in Database:**
- `BlueSkyChat` model with conversation metadata

---

### 2.4 Get Messages

Fetches messages for a specific chat conversation.

**Method:** `BlueSkyService.get_messages_for_chat(account, chat, access_token, is_archive)`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account` | Account | Account model instance |
| `chat` | BlueSkyChat | Chat model instance |
| `access_token` | string | OAuth access token |
| `is_archive` | boolean | True for full archive, False for incremental |

**BlueSky API Called (via PDS proxy):**
```
GET {pds_base}/xrpc/chat.bsky.convo.getMessages
```

**Required Headers:**
```http
Authorization: DPoP <access_token>
DPoP: <dpop_proof>
Atproto-Proxy: did:web:api.bsky.chat#bsky_chat
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `convoId` | string | Conversation ID |
| `limit` | integer | Number of messages per page (default: 100) |
| `cursor` | string | Pagination cursor |

**Response Data Structure:**
```json
{
    "messages": [
        {
            "$type": "chat.bsky.convo.defs#messageView",
            "id": "msg123",
            "rev": "rev123",
            "text": "Message content",
            "sentAt": "2024-01-15T12:00:00.000Z",
            "sender": {
                "did": "did:plc:xxx"
            },
            "embed": {
                "$type": "app.bsky.embed.record#view",
                "record": {}
            }
        }
    ],
    "cursor": "next_page_cursor"
}
```

**Stored in Database:**
- `BlueSkyMessage` model with message content and metadata

---

### 2.5 Archive

Generates archive export of BlueSky data.

**Method:** `BlueSkyService.archive(user_id, account_id)`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | integer | User ID |
| `account_id` | integer | Account ID |

**Output:**
- CSV files for posts
- PDF exports
- ZIP archive uploaded to S3

---

## 3. External BlueSky API Endpoints Used

### Authentication

| Endpoint | Method | Description |
|----------|--------|-------------|
| `https://bsky.social/.well-known/oauth-authorization-server` | GET | OAuth metadata |
| `https://bsky.social/oauth/par` | POST | Pushed Authorization Request |
| `https://bsky.social/oauth/authorize` | GET | User authorization |
| `https://bsky.social/oauth/token` | POST | Token exchange/refresh |
| `{pds_base}/oauth/revoke` | POST | Token revocation |

### AT Protocol

| Endpoint | Method | Description |
|----------|--------|-------------|
| `{pds_base}/xrpc/com.atproto.server.getSession` | GET | Get session info |

### BlueSky AppView (Public API)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `https://public.api.bsky.app/xrpc/app.bsky.actor.getProfile` | GET | Get user profile |
| `https://public.api.bsky.app/xrpc/app.bsky.feed.getAuthorFeed` | GET | Get user posts |
| `https://public.api.bsky.app/xrpc/app.bsky.feed.getPostThread` | GET | Get post thread |

### BlueSky Chat API (via PDS Proxy)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `{pds_base}/xrpc/chat.bsky.convo.listConvos` | GET | List conversations |
| `{pds_base}/xrpc/chat.bsky.convo.getMessages` | GET | Get messages |
| `{pds_base}/xrpc/chat.bsky.convo.sendMessage` | POST | Send message |

**Note:** Chat API requests require `Atproto-Proxy: did:web:api.bsky.chat#bsky_chat` header.

---

## 4. Error Codes

### Internal API Error Codes

| Status | Code | Description |
|--------|------|-------------|
| 200 | - | Success |
| 302 | - | Redirect (OAuth flow) |
| 400 | - | General error |
| 401 | `access_denied` | Authentication denied by user |
| 402 | - | Authentication failed |
| 403 | - | Invalid guest token |
| 404 | - | Token expired / Account not found |
| 405 | - | Permission denied |
| 406 | - | No authentication credentials |
| 407 | - | Token exchange failed |
| 408 | - | Session fetch failed |
| 500 | - | Internal server error |

### BlueSky API Error Codes

| Error | Description |
|-------|-------------|
| `invalid_request` | Malformed request |
| `invalid_client` | Client authentication failed |
| `invalid_grant` | Authorization code invalid/expired |
| `invalid_token` | Token invalid or revoked |
| `use_dpop_nonce` | DPoP nonce required - retry with nonce |
| `ExpiredToken` | Access token expired |

---

## 5. Data Models

### Account

```python
{
    "id": 123,
    "provider": "Bluesky",
    "uid": "did:plc:xxx",            # BlueSky DID
    "name": "Display Name",
    "username": "handle.bsky.social",
    "token": "refresh_token",
    "token_secret": "",
    "profile_image_url": "https://...",
    "user_id": 1,
    "date_joined": "2024-01-15T12:00:00Z",
    "last_login": "2024-01-15T12:00:00Z",
    "date_closed": null,
    "status": null
}
```

### Bluesky (Post)

```python
{
    "id": 456,
    "user_id": 1,
    "account_id": 123,
    "social_id": "3abc123xyz",        # Post rkey
    "uri": "at://did:plc:xxx/app.bsky.feed.post/xxx",
    "cid": "bafyreiabc123...",
    "text": "Post content",
    "created_at": "2024-01-15T12:00:00Z",
    "reply_count": 5,
    "repost_count": 10,
    "like_count": 25,
    "media": [...],
    "embed": {...},
    "facets": [...],
    "is_reply": false,
    "is_repost": false,
    "parent_uri": null
}
```

### BlueSkyChat

```python
{
    "id": 789,
    "user_id": 1,
    "account_id": 123,
    "convo_id": "3abc123xyz",
    "status": "allowed",
    "participants": [
        {
            "did": "did:plc:xxx",
            "handle": "user.bsky.social",
            "displayName": "User Name",
            "avatar": "https://..."
        }
    ],
    "unread_count": 0,
    "message_count": 15,
    "muted": false,
    "last_message": {...},
    "created_at": "2024-01-15T12:00:00Z",
    "updated_at": "2024-01-15T12:00:00Z"
}
```

### BlueSkyMessage

```python
{
    "id": 101,
    "user_id": 1,
    "account_id": 123,
    "chat_id": 789,
    "message_id": "msg123",
    "text": "Message content",
    "sender_did": "did:plc:xxx",
    "sender_handle": "user.bsky.social",
    "sender_name": "User Name",
    "sent_at": "2024-01-15T12:00:00Z",
    "embed": {...},
    "is_deleted": false
}
```

---

## Configuration

### Environment Variables

| Variable | Description |
|----------|-------------|
| `SOCIAL_BLUESKY_CLIENT_ID` | OAuth Client ID (usually callback URL) |
| `SOCIAL_BLUESKY_REDIRECT` | OAuth redirect/callback URL |
| `SOCIAL_BLUESKY_SCOPE` | OAuth scopes (if overriding defaults) |

### Client Metadata URL

The client ID must resolve to a JSON document at:
```
{SOCIAL_BLUESKY_CLIENT_ID}/.well-known/oauth-client-metadata
```

Example client metadata:
```json
{
    "client_id": "https://api.sharparchive.com/bluesky/callback",
    "client_name": "SharpArchive",
    "client_uri": "https://sharparchive.com",
    "redirect_uris": ["https://api.sharparchive.com/bluesky/callback"],
    "scope": "atproto transition:generic transition:chat.bsky",
    "grant_types": ["authorization_code", "refresh_token"],
    "response_types": ["code"],
    "token_endpoint_auth_method": "none",
    "dpop_bound_access_tokens": true
}
```

---

## Security Notes

1. **DPoP Tokens**: All authenticated requests use DPoP (Demonstrating Proof-of-Possession) tokens. Keys are stored in `AccountMeta` for persistence across requests.

2. **PKCE**: OAuth flow uses PKCE (Proof Key for Code Exchange) with S256 challenge method.

3. **Key Binding**: Access tokens are bound to DPoP keys via JKT (JSON Web Key Thumbprint) verification.

4. **Token Refresh**: Refresh tokens may rotate on each use. New tokens are automatically persisted.

5. **Session Cleanup**: OAuth session data is cleaned up after successful authentication.

---

*Last Updated: January 2026*
*Version: DEV-SA-45-BLUESKY-INTEGRATION*
