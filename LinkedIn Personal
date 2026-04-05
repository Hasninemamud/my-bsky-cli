# LinkedIn Personal (`linkedin_personal`)

This Django app archives a **personal LinkedIn** profile’s posts and comments by connecting the user through **UniPile** hosted authentication. Data is stored in app-specific models, exposed via REST and legacy JSON social endpoints, and can be exported through the archive pipeline.

> **Scope:** This repository is the **SharpArchive API** (Django). A separate web client consumes these endpoints. The “frontend” section below describes that contract and the popup/localStorage pattern implemented in this repo’s templates.

---

## High-level flow

```mermaid
sequenceDiagram
    participant User
    participant Client as Web client
    participant API as SharpArchive API
    participant UniPile as UniPile API
    participant Celery as Celery workers

    User->>Client: Connect LinkedIn Personal
    Client->>API: GET auth-url (or open login flow)
    API->>UniPile: POST hosted/accounts/link
    UniPile-->>API: auth URL
    API-->>Client: auth_url
    User->>UniPile: OAuth / hosted login
    UniPile->>API: Redirect to .../authorized/?code=...
    API->>UniPile: GET account + users/me
    API->>API: Create/update social_app.Account
    API->>Celery: sync_linkedin_posts_and_comments_task
    Celery->>UniPile: Fetch posts & comments
    Celery->>API: Persist content, attachments, members
    Celery->>API: archive_linkedin_personal_account_task
```

---

## Backend architecture

| Layer | Role |
|--------|------|
| **`api/v1/views/`** | REST: integration (list/connect), hosted-auth URL, browser login/callback, content/comments (used internally from `article.py`) |
| **`services/auth_service.py`** | UniPile client calls: hosted link, account details, `users/me`, user posts, post comments; `process_authorization`; disconnect |
| **`services/content_service.py`** | Sync posts/comments into DB, members, attachments; list/filter APIs for content and comments |
| **`services/linkedin_attachment_service.py`** | Download media/profile images and store via archive API URLs |
| **`services/linkedin_archive_service.py`** | Build CSV exports + S3 archive snapshot for an account |
| **`helpers/client.py`** | HTTP client for UniPile (`X-API-KEY`, `/api/v1/...`) |
| **`repositories/`** | Thin persistence for members, content, comments, attachments |
| **`models/`** | `LinkedInPersonalMember`, `LinkedInPersonalContent`, `LinkedInPersonalComment`, `LinkedInPersonalContentAttachment` |
| **`tasks.py`** | Celery: per-account sync, sync-all, per-account and global archive jobs |

### Provider identity

- Enum: `Providers.LINKEDIN_PERSONAL` → string value **`LinkedInPersonal`** (used in JSON from social views and when filtering `Account` rows).
- Linked accounts are normal `social_app.Account` rows with `provider=LinkedInPersonal`. The **UniPile account id** is stored in `Account.token` (access token field reused as external id).

### UniPile integration

Configured via **`UNIPILE_API_KEY`** and **`UNIPILE_API_URL`** (see `my_settings.py`). The client calls versioned paths such as:

- `POST hosted/accounts/link` — create hosted auth link (`type: create` or `reconnect`)
- `GET accounts/{id}` — account metadata after callback
- `GET users/me?account_id=...` — profile fields, counters, `provider_id`, etc.
- `GET users/{identifier}/posts` — feed pages (cursor pagination)
- `GET posts/{post_id}/comments` — comment pages

Constants live in `constants.py` (`UNIPILE_*`).

### Authorization outcomes

1. **`LinkedInPersonalAuthAPIView` (`GET …/auth-url/`)**  
   Authenticated API user. Builds success/failure redirects (defaults from `SOCIAL_LINKEDIN_PERSONAL_REDIRECT_URL` and notify `SOCIAL_LINKEDIN_PERSONAL_NOTIFY_URL`). Returns `{ "data": { "auth_url": "..." } }`.

2. **`LinkedInIntegrationAPIView`**
   - **`GET`** — list connected accounts for the user (`Account` with `provider=LinkedInPersonal`, not closed).
   - **`POST`** — body/query `code` + optional `state`; runs `process_authorization` (same persistence as callback). Used when the client completes the flow via API instead of the browser template.

3. **`LinkedInLoginView` (`GET|POST …/login/`)**  
   Django session user. Builds redirect URIs from named routes `linkedin_personal_authorized` and `linkedin_personal_notify`. Returns JSON: `{ "status": 302, "Location": "<unipile url>" }` or error — **clients must perform the redirect** (not an HTTP 302 response).

4. **`LinkedInBackAuthorizedView` (`GET …/authorized/`)**  
   Browser callback: validates error query params, reads `code` or `account_id`, calls `process_authorization`, queues `sync_linkedin_posts_and_comments_task`, renders `PROFILE_POPUP_CLOSE_TEMPLATE` with JSON payload for the opener window.

### `process_authorization` (summary)

- Resolves SharpArchive user (and optional guest) from request + `state` via `resolve_user_from_request`.
- Loads UniPile account + `users/me` to set name, username, profile image, metadata, follower/connection counts.
- Creates or updates `Account` via `AccountService`; sets `type` to personal; stores `metadata` from UniPile profile.
- Optionally archives profile image through `ARCHIVE_API_URL`.
- Deletes a previous UniPile account if the same LinkedIn username was already linked (reconnection hygiene).
- Fires **`sync_linkedin_posts_and_comments_task.delay(account.id)`** immediately after connect.

### Sync and storage

- **`sync_linkedin_posts_and_comments_task`** — loads posts in pages (`max_pages=10`, `limit=100`), upserts `LinkedInPersonalContent`, syncs attachments, then **`sync_post_comments`** per post. Triggers **`archive_linkedin_personal_account_task`** after sync.
- **`sync_all_linkedin_posts_and_comments_task`** — scheduled (see `SharpArchive/celery_config.py`) to queue sync for every LinkedIn Personal account.
- **`archive_linkedin_personal_task`** — scheduled bulk archive for all accounts.
- Content lists for the UI use **`post__isnull=True`** on `LinkedInPersonalContent` so rows that represent comments linked to a post are excluded from the main post feed query.

---

## HTTP routes (this app)

Mounted under **`/api/v1/linkedin-personal/`** (see `api_app/urls.py`).

| Method | Path | View | Notes |
|--------|------|------|--------|
| GET | `/` | `LinkedInIntegrationAPIView` | List accounts; requires authenticated user for success |
| POST | `/` | `LinkedInIntegrationAPIView` | Connect: `code` (+ optional `state`) |
| GET | `/auth-url/` | `LinkedInPersonalAuthAPIView` | Returns UniPile `auth_url`; query: `state`, optional `redirect_url` |
| GET/POST | `/login/` | `LinkedInLoginView` | Session-based; JSON with `Location` for redirect |
| GET | `/authorized/` | `LinkedInBackAuthorizedView` | OAuth-style callback; HTML popup closer |

`LinkedInContentAPIView` and `LinkedInCommentAPIView` are **not** wired in `api/v1/urls.py`; they are invoked from **`api_app.views.social.article`** (see below).

---

## Integration with legacy “social” JSON API

These routes are under **`/api/social/`** and power the main archive UI.

| Endpoint | LinkedIn Personal behavior |
|----------|----------------------------|
| **`GET social/article/`** | If `account.provider == LinkedInPersonal`, delegates to `LinkedInContentAPIView.get` for items + pagination; sets `earliestDate` from `LinkedInPersonalContentService.get_earliest_date`. Requires `accountId`. |
| **`GET social/article-more/`** | Same content view for pagination (`lastItemId`, date range, etc.). |
| **`POST social/comment/`** | For `provider == LinkedInPersonal`, sets `postId` on the request and calls `LinkedInCommentAPIView.get`. Request body must use provider string **`LinkedInPersonal`** (same as `Providers.LINKEDIN_PERSONAL.value`). |
| **`GET social/earliest-date/`** | Includes LinkedIn Personal in global earliest-content calculation. |
| **`GET social/feed/`** (and related) | Uses `resolve_account_profile_pic` for LinkedIn Personal profile images; **DELETE** on feed disconnect calls `LinkedinPersonalAuthService.disconnect_account` (UniPile delete + clear tokens). |

When articles are loaded for an account with no local data yet, **`get_user_articles`** queues **`sync_linkedin_posts_and_comments_task.delay(account.id)`** for LinkedIn Personal.

---

## Frontend (client app) expectations

This codebase does not ship the SPA, but the contract is:

1. **Authentication**  
   Use normal SharpArchive session/JWT as for other social providers.

2. **Starting LinkedIn Personal connect**
   - **Option A:** `GET /api/v1/linkedin-personal/auth-url/?state=...&redirect_url=...` then open `auth_url` in a window/tab.
   - **Option B:** `GET|POST /api/v1/linkedin-personal/login/` with session cookie, read JSON `Location`, redirect the user to UniPile.

3. **After UniPile redirects**  
   User lands on `/api/v1/linkedin-personal/authorized/`. Response is HTML (`templates/back/pages/settings/profile_popup_close.html`) that runs:

   - `localStorage.setItem('popupProvider', …)`
   - `localStorage.setItem('popupData', …)` — JSON with `state`, `message`, and on success `provider`, `account_id`
   - `localStorage.setItem('popupClosed', 'true')`
   - `window.close()`

   The opener should **poll `localStorage`** (or use `storage` events) to detect completion, same pattern as other OAuth popups.

4. **Feed and content**  
   Use existing social endpoints with **`provider: "LinkedInPersonal"`** where a provider string is required (e.g. comment POST body).

5. **Profile images**  
   URLs may point to archived copies; `resolve_account_profile_pic` normalizes display for feed/article responses.

---

## Data model (short)

- **`LinkedInPersonalMember`** — authors and commenters (provider ids, names, archived avatars).
- **`LinkedInPersonalContent`** — posts (`content_type` post) with counters, `metadata` JSON, optional `reposted_by`; self-FK for parent/repost structure; `post` FK links comment-rows to a parent post where applicable.
- **`LinkedInPersonalComment`** — comments tied to `post` + `author`.
- **`LinkedInPersonalContentAttachment`** — files/thumbnails for a post, optionally archived.

---

## Configuration checklist

Set in environment / `my_settings.py` (names only):

- `UNIPILE_API_KEY`, `UNIPILE_API_URL`
- `SOCIAL_LINKEDIN_PERSONAL_REDIRECT_URL` — default success redirect for hosted auth from `auth-url`
- `SOCIAL_LINKEDIN_PERSONAL_NOTIFY_URL` — webhook base (often on archive API)
- `ARCHIVE_API_URL` — media and profile archiving
- Celery broker/worker enabled for sync and archive tasks

---

## Related code

- **Moderation / content viewer:** `apps/moderation/services/content_viewer.py` serializes LinkedIn Personal posts and comments.
- **Search:** `api_app/services/provider_search_service.py` searches LinkedIn Personal content.
- **Downloads:** `api_app/services/download_service.py` references `LinkedInPersonalContent`.

---

## File map

```
apps/platforms/linkedin_personal/
├── api/v1/
│   ├── urls.py
│   ├── views/
│   │   ├── linkedin_auth_view.py    # auth-url, login, authorized callback
│   │   └── linkedin_integration_view.py  # integration, content/comment views
│   └── serializers/
├── constants.py
├── enum.py
├── helpers/client.py
├── models/
├── repositories/
├── services/
│   ├── auth_service.py
│   ├── content_service.py
│   ├── linkedin_attachment_service.py
│   └── linkedin_archive_service.py
├── tasks.py
└── README.md
```
