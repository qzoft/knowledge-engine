# Feature Plan: Authentication & Authorization

**Priority**: High  
**Depends On**: None (extends existing web layer)  
**Foundation For**: Multi-Reviewer Collaboration (#10), Discussion Harvesting (#11) — provides user identity for all write operations  
**Theory**: Principle of least privilege — read access is public, write access requires identity

---

## Overview

Add authentication to the web UI and REST API. Unauthenticated users can browse and search (read-only). Authenticated users can edit documents, change status, review, approve, and use admin features. The system supports two modes: **local accounts** (with sign-up) and **SSO** (OpenID Connect / OAuth2), configurable in `rag-config.yaml`. The authenticated username is used as the identity for all metadata operations (`author`, `approved_by`, `in_review_by`, contributor tracking).

---

## 1. Configuration — `config.py` / `rag-config.yaml`

### Config Schema

```yaml
auth:
  enabled: true
  mode: "local"          # "local" | "sso"
  
  # Local mode settings
  local:
    allow_signup: true    # Allow new account registration
    
  # SSO mode settings (OpenID Connect)
  sso:
    provider_url: "https://login.example.com"
    client_id: "rag-knowledge-engine"
    client_secret: "${RAG_SSO_SECRET}"   # env var reference
    scopes: ["openid", "profile", "email"]
    # Claim mapping
    username_claim: "preferred_username"
    email_claim: "email"
    display_name_claim: "name"
```

### Config Dataclass

```python
@dataclass
class AuthConfig:
    enabled: bool = False
    mode: str = "local"             # "local" | "sso"
    allow_signup: bool = True       # Only for local mode
    provider_url: str = ""          # SSO: OIDC discovery endpoint
    client_id: str = ""             # SSO: client ID
    client_secret: str = ""         # SSO: client secret (supports ${ENV_VAR})
    scopes: list[str] = field(default_factory=lambda: ["openid", "profile", "email"])
    username_claim: str = "preferred_username"
    email_claim: str = "email"
    display_name_claim: str = "name"
    session_secret: str = ""        # Auto-generated if empty
    session_max_age: int = 86400    # 24 hours
```

### Behavior When Disabled

When `auth.enabled` is `false` (default):
- Everything works as today — no login, full access
- All write operations use `author: "Local"` (current behavior)
- No breaking change for existing deployments

---

## 2. Database Schema — `fts_store.py`

### New Table: `users` (local mode only)

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    email TEXT NOT NULL UNIQUE,
    display_name TEXT NOT NULL DEFAULT '',
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'editor',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    last_login TEXT,
    is_active BOOLEAN NOT NULL DEFAULT 1
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_users_username ON users(username);
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email);
```

### Role Hierarchy

| Role | Can View | Can Edit | Can Review/Approve | Can Admin |
|------|----------|----------|-------------------|-----------|
| `viewer` | Yes | No | No | No |
| `editor` | Yes | Yes | No | No |
| `reviewer` | Yes | Yes | Yes | No |
| `admin` | Yes | Yes | Yes | Yes |

Note: Unauthenticated users have implicit `viewer` access when auth is enabled.

### Password Hashing

Use `argon2-cffi` (memory-hard hashing, recommended over bcrypt):

```python
from argon2 import PasswordHasher

ph = PasswordHasher()
hash = ph.hash(password)
ph.verify(hash, password)  # raises on mismatch
```

---

## 3. FTS Store Methods — `fts_store.py`

### User Management (local mode)

```python
def create_user(self, username: str, email: str, password: str,
                display_name: str = "", role: str = "editor") -> dict:
    """Create a new user account. Hashes password with argon2. Returns user dict (no password)."""

def authenticate_user(self, username: str, password: str) -> dict | None:
    """Verify credentials. Returns user dict on success, None on failure. Updates last_login."""

def get_user(self, username: str) -> dict | None:
    """Get user by username (excludes password hash)."""

def get_user_by_email(self, email: str) -> dict | None:
    """Get user by email (excludes password hash)."""

def update_user_role(self, username: str, role: str) -> bool:
    """Change a user's role. Admin only."""

def list_users(self) -> list[dict]:
    """List all users (excludes password hashes)."""

def deactivate_user(self, username: str) -> bool:
    """Soft-delete a user (set is_active=False)."""
```

### SSO User Sync

```python
def upsert_sso_user(self, username: str, email: str, display_name: str) -> dict:
    """Create or update a user from SSO claims. No password stored. Returns user dict."""
```

---

## 4. Authentication Middleware — `web/auth.py` (new file)

### Session Management

Use Starlette's `SessionMiddleware` with signed cookies:

```python
from starlette.middleware.sessions import SessionMiddleware

app.add_middleware(
    SessionMiddleware,
    secret_key=config.auth.session_secret,
    max_age=config.auth.session_max_age,
    https_only=False,  # Allow HTTP for local dev
    same_site="lax",
)
```

### Auth Middleware

```python
class AuthMiddleware:
    """Middleware that sets request.state.user from session."""
    
    async def __call__(self, scope, receive, send):
        # Read user from session cookie
        # Set request.state.user = {username, role, display_name} or None
        # Continue to next middleware/route
```

### Route Protection

```python
def require_auth(min_role: str = "editor"):
    """Decorator for routes that require authentication."""
    # Check request.state.user exists
    # Check user.role >= min_role
    # Return 401 (not logged in) or 403 (insufficient role)

def get_current_user(request) -> dict | None:
    """Extract current user from request. Returns None if not authenticated."""
```

### Permission Checks

| Operation | Minimum Role |
|-----------|-------------|
| Browse / Search / Read | None (public) |
| Edit document content | `editor` |
| Change document status | `reviewer` |
| Approve documents | `reviewer` |
| Admin panel (reindex, export, user management) | `admin` |
| Create new documents via web | `editor` |
| Sign up (local mode) | None |

---

## 5. Auth Routes — `web/app.py`

### Local Mode Routes

| Route | Method | Description |
|-------|--------|-------------|
| `GET /login` | GET | Login form |
| `POST /login` | POST | Authenticate and create session |
| `GET /signup` | GET | Registration form (if `allow_signup=true`) |
| `POST /signup` | POST | Create account and auto-login |
| `POST /logout` | POST | Clear session |
| `GET /profile` | GET | User profile page |

### SSO Routes

| Route | Method | Description |
|-------|--------|-------------|
| `GET /login` | GET | Redirect to SSO provider |
| `GET /auth/callback` | GET | Handle SSO callback, create session |
| `POST /logout` | POST | Clear session (optionally redirect to SSO logout) |

### REST API Auth

| Route | Method | Description |
|-------|--------|-------------|
| `POST /api/auth/login` | POST | Authenticate, return session token |
| `POST /api/auth/signup` | POST | Register (local mode, if enabled) |
| `POST /api/auth/logout` | POST | Clear session |
| `GET /api/auth/me` | GET | Current user info |
| `GET /api/admin/users` | GET | List users (admin only) |
| `PATCH /api/admin/users/{username}/role` | PATCH | Change user role (admin only) |

---

## 6. Identity Integration — Existing Features

### Document Editing

When an authenticated user edits a document via the web UI or API, use `request.state.user.username` as the `author`:

```python
# In document content update endpoint
user = get_current_user(request)
result = indexer.update_document(
    file_path=file_path,
    content=content,
    author=user["username"] if user else "Anonymous",
)
```

### Status Changes

When changing status to `in_review`, set `in_review_by` to the authenticated username.
When changing status to `approved`, set `approved_by` to the authenticated username.

### Document Creation

`save_knowledge` via web uses the authenticated username as default author.

### MCP Tool Identity

MCP tools run via stdio (not HTTP), so they don't have web sessions. MCP tool calls continue to accept `author` as a parameter. The web UI provides identity automatically; MCP clients provide it explicitly.

---

## 7. Web UI Changes

### Navigation Bar

Add auth state to the base template:

```html
<nav class="nav">
    <!-- existing nav items -->
    {% if user %}
        <span class="nav__user">{{ user.display_name or user.username }}</span>
        <span class="nav__role">({{ user.role }})</span>
        <form method="post" action="/logout" class="nav__logout">
            <button type="submit">Logout</button>
        </form>
    {% elif auth_enabled %}
        <a href="/login" class="nav__login">Login</a>
    {% endif %}
</nav>
```

### Read-Only Mode for Unauthenticated Users

When auth is enabled and user is not logged in:
- Hide edit buttons on document pages
- Hide status change dropdowns
- Hide "New Document" buttons
- Show a subtle "Login to edit" prompt
- Admin panel redirects to login

### Login Page

Clean, minimal login form matching the terminal aesthetic. Two fields (username/email + password), login button, optional "Sign up" link.

### Sign-Up Page (local mode)

Four fields: username, email, display name (optional), password. Clear password requirements. Auto-login after registration.

### Admin User Management

New section in admin panel (admin role only):
- User list with username, email, role, last login, active status
- Role change dropdown per user
- Deactivate/reactivate toggle

---

## 8. Dependencies

### New Python Dependencies

| Package | Purpose |
|---------|---------|
| `argon2-cffi` | Password hashing (local mode) |
| `authlib` | OIDC/OAuth2 client (SSO mode, optional) |
| `itsdangerous` | Session signing (used by Starlette's SessionMiddleware) |

`authlib` is only needed when SSO mode is configured. It should be an optional dependency:

```toml
[project.optional-dependencies]
sso = ["authlib>=1.3"]
```

---

## 9. Security Considerations

- **Password storage**: Argon2id with default parameters (memory-hard, resistant to GPU attacks)
- **Session cookies**: Signed with `session_secret`, `SameSite=Lax`, `HttpOnly=True`
- **CSRF protection**: Starlette's middleware handles CSRF for form submissions
- **Rate limiting**: Login endpoint should have basic rate limiting (e.g., 5 attempts per minute per IP)
- **SSO secrets**: Support `${ENV_VAR}` syntax in config to avoid secrets in YAML files
- **First user**: When local mode is enabled and no users exist, the first sign-up gets `admin` role automatically

---

## Implementation Order

| Phase | Scope |
|-------|-------|
| **Phase 1** | Config schema, `users` table, user CRUD methods, password hashing |
| **Phase 2** | Session middleware, auth middleware, login/logout routes (local mode) |
| **Phase 3** | Sign-up flow, first-user admin promotion, route protection |
| **Phase 4** | Identity integration — author/reviewer auto-fill from session |
| **Phase 5** | Admin user management UI |
| **Phase 6** | SSO mode (OIDC discovery, callback, user sync) |
