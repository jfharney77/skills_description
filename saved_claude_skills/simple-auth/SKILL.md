# /simple-auth

Add username/password authentication to a FastAPI + React (Vite) monorepo. Produces JWT-based auth with an admin role and self-registration. All wikis/data remain shared across authenticated users.

## What this skill adds

- `POST /api/auth/register` — self-registration (creates `user` role)
- `POST /api/auth/login` → JWT token
- `GET /api/auth/me` — validate token, return current user info
- `GET/POST /api/admin/users` — list and create users (admin only)
- `PATCH /api/admin/users/{id}/role` — promote/demote (admin only)
- `DELETE /api/admin/users/{id}` — remove user (admin only)
- All existing endpoints protected by `Depends(get_current_user)`
- Bootstrap: first admin created from env vars on startup if no users exist
- Frontend: login/register page gates the entire app; Admin tab for admins; logout button in topbar

## Audit first

Before making changes, check:
- Does `backend/auth.py` exist with `hash_password`, `verify_password`, `create_token`, `decode_token`?
- Does `backend/db.py` have a `users` table and user CRUD (`create_user`, `get_user_by_username`, `list_users`, `update_user_role`, `delete_user`, `count_users`)?
- Does `backend/main.py` import `auth`, define `get_current_user` / `require_admin` dependencies, and add them to all endpoints?
- Does `frontend/src/api.js` exist with `getToken`, `setToken`, `apiFetch`, `streamUrl`?
- Do all frontend components import from `api.js` instead of calling `fetch()` directly?
- Does `frontend/src/components/LoginPage.jsx` exist?
- Does `frontend/src/components/AdminPanel.jsx` exist?

Report what is already in place before making changes.

## 1. Python dependencies

Add to `backend/requirements.txt`:
```
passlib[bcrypt]
python-jose[cryptography]
```

## 2. backend/auth.py (new file)

```python
import os
from datetime import datetime, timedelta, timezone
from passlib.context import CryptContext
from jose import jwt, JWTError

_pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")
_SECRET = os.getenv("JWT_SECRET", "dev-secret-change-in-production")
_ALGO = "HS256"
_EXPIRE_HOURS = int(os.getenv("JWT_EXPIRE_HOURS", "24"))

def hash_password(password: str) -> str:
    return _pwd.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return _pwd.verify(plain, hashed)

def create_token(user_id: int, username: str, role: str) -> str:
    exp = datetime.now(timezone.utc) + timedelta(hours=_EXPIRE_HOURS)
    return jwt.encode(
        {"sub": str(user_id), "username": username, "role": role, "exp": exp},
        _SECRET, algorithm=_ALGO,
    )

def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, _SECRET, algorithms=[_ALGO])
    except JWTError as e:
        raise ValueError(str(e))
```

## 3. backend/db.py — add users table + CRUD

Add to `init_pool` (inside the `CREATE TABLE IF NOT EXISTS` block):
```sql
CREATE TABLE IF NOT EXISTS users (
    id            SERIAL PRIMARY KEY,
    username      TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'user',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
)
```

Add these functions to `db.py`:
```python
async def create_user(username: str, password_hash: str, role: str = "user") -> dict:
    async with get_pool().acquire() as conn:
        try:
            row = await conn.fetchrow(
                "INSERT INTO users (username, password_hash, role) VALUES ($1, $2, $3) "
                "RETURNING id, username, role, created_at",
                username, password_hash, role,
            )
            return dict(row)
        except asyncpg.UniqueViolationError:
            raise ValueError(f"Username '{username}' is already taken.")

async def get_user_by_username(username: str) -> dict | None:
    async with get_pool().acquire() as conn:
        row = await conn.fetchrow(
            "SELECT id, username, password_hash, role, created_at FROM users WHERE username=$1", username
        )
        return dict(row) if row else None

async def list_users() -> list[dict]:
    async with get_pool().acquire() as conn:
        rows = await conn.fetch("SELECT id, username, role, created_at FROM users ORDER BY created_at")
        return [dict(r) for r in rows]

async def update_user_role(user_id: int, role: str) -> dict | None:
    async with get_pool().acquire() as conn:
        row = await conn.fetchrow(
            "UPDATE users SET role=$1 WHERE id=$2 RETURNING id, username, role, created_at", role, user_id
        )
        return dict(row) if row else None

async def delete_user(user_id: int) -> bool:
    async with get_pool().acquire() as conn:
        result = await conn.execute("DELETE FROM users WHERE id=$1", user_id)
        return result == "DELETE 1"

async def count_users() -> int:
    async with get_pool().acquire() as conn:
        return await conn.fetchval("SELECT COUNT(*) FROM users")
```

## 4. backend/main.py — auth dependencies + endpoints

Add imports:
```python
from fastapi import Depends, Header
from fastapi import Query as QParam
import auth
```

Add dependency functions:
```python
async def get_current_user(
    authorization: str = Header(None),
    token: str = QParam(None),
) -> dict:
    # token query param is the SSE fallback — EventSource can't send headers
    t = None
    if authorization and authorization.startswith("Bearer "):
        t = authorization[7:]
    elif token:
        t = token
    if not t:
        raise HTTPException(status_code=401, detail="Not authenticated")
    try:
        return auth.decode_token(t)
    except ValueError:
        raise HTTPException(status_code=401, detail="Invalid or expired token")

async def require_admin(user: dict = Depends(get_current_user)) -> dict:
    if user.get("role") != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user
```

Add to `lifespan` (after DB init):
```python
first_admin = os.getenv("FIRST_ADMIN_USERNAME", "").strip()
first_pass  = os.getenv("FIRST_ADMIN_PASSWORD", "").strip()
if first_admin and first_pass and await db.count_users() == 0:
    await db.create_user(first_admin, auth.hash_password(first_pass), role="admin")
    print(f"[app] Created initial admin: {first_admin}")
```

Add auth endpoints (public — no Depends):
```python
class RegisterRequest(BaseModel):
    username: str
    password: str

class LoginRequest(BaseModel):
    username: str
    password: str

@app.post("/api/auth/register", status_code=201)
async def register(req: RegisterRequest):
    if len(req.username.strip()) < 2:
        raise HTTPException(400, "Username must be at least 2 characters.")
    if len(req.password) < 6:
        raise HTTPException(400, "Password must be at least 6 characters.")
    try:
        user = await db.create_user(req.username.strip(), auth.hash_password(req.password))
        return {"id": user["id"], "username": user["username"], "role": user["role"]}
    except ValueError as e:
        raise HTTPException(409, str(e))

@app.post("/api/auth/login")
async def login(req: LoginRequest):
    user = await db.get_user_by_username(req.username.strip())
    if not user or not auth.verify_password(req.password, user["password_hash"]):
        raise HTTPException(401, "Invalid username or password.")
    return {"token": auth.create_token(user["id"], user["username"], user["role"]),
            "username": user["username"], "role": user["role"]}

@app.get("/api/auth/me")
async def me(user: dict = Depends(get_current_user)):
    return {"id": user["sub"], "username": user["username"], "role": user["role"]}
```

Add admin endpoints:
```python
class AdminCreateUserRequest(BaseModel):
    username: str
    password: str
    role: str = "user"

class UpdateRoleRequest(BaseModel):
    role: str

@app.get("/api/admin/users")
async def admin_list_users(_admin: dict = Depends(require_admin)):
    return {"users": await db.list_users()}

@app.post("/api/admin/users", status_code=201)
async def admin_create_user(req: AdminCreateUserRequest, _admin: dict = Depends(require_admin)):
    if req.role not in ("admin", "user"):
        raise HTTPException(400, "Role must be 'admin' or 'user'.")
    try:
        user = await db.create_user(req.username.strip(), auth.hash_password(req.password), req.role)
        return {"id": user["id"], "username": user["username"], "role": user["role"]}
    except ValueError as e:
        raise HTTPException(409, str(e))

@app.patch("/api/admin/users/{user_id}/role")
async def admin_update_role(user_id: int, req: UpdateRoleRequest, admin: dict = Depends(require_admin)):
    if req.role not in ("admin", "user"):
        raise HTTPException(400, "Role must be 'admin' or 'user'.")
    if str(user_id) == admin.get("sub"):
        raise HTTPException(400, "Cannot change your own role.")
    user = await db.update_user_role(user_id, req.role)
    if not user:
        raise HTTPException(404, "User not found.")
    return user

@app.delete("/api/admin/users/{user_id}")
async def admin_delete_user(user_id: int, admin: dict = Depends(require_admin)):
    if str(user_id) == admin.get("sub"):
        raise HTTPException(400, "Cannot delete your own account.")
    if not await db.delete_user(user_id):
        raise HTTPException(404, "User not found.")
    return {"message": "User deleted."}
```

Add `_user: dict = Depends(get_current_user)` to every existing endpoint. Keep `GET /health` public.

## 5. frontend/src/api.js (new file)

```js
const BASE = import.meta.env.VITE_API_URL ?? ''

export function getToken() { return localStorage.getItem('app_token') }
export function setToken(t) {
  if (t) localStorage.setItem('app_token', t)
  else localStorage.removeItem('app_token')
}

export async function apiFetch(method, path, body = null) {
  const token = getToken()
  const isFormData = body instanceof FormData
  const headers = {}
  if (!isFormData && body !== null) headers['Content-Type'] = 'application/json'
  if (token) headers['Authorization'] = `Bearer ${token}`
  const res = await fetch(`${BASE}${path}`, {
    method, headers,
    body: isFormData ? body : (body !== null ? JSON.stringify(body) : null),
  })
  if (res.status === 401) { setToken(null); window.dispatchEvent(new Event('auth:logout')) }
  return res
}

// SSE: EventSource can't send headers — pass token as ?token= for the stream endpoint only.
export function streamUrl(path) {
  const token = getToken()
  if (!token) return `${BASE}${path}`
  const sep = path.includes('?') ? '&' : '?'
  return `${BASE}${path}${sep}token=${encodeURIComponent(token)}`
}
```

Replace every `fetch(${API}/path)` call in components with `apiFetch('METHOD', '/path', body?)`.
Replace every `new EventSource(${API}/path)` with `new EventSource(streamUrl('/path'))`.
Remove `const API = import.meta.env.VITE_API_URL ?? ''` from each component.

## 6. frontend/src/components/LoginPage.jsx (new)

A card with Sign In / Register tabs. On register, auto-login after success. On login, call `setToken(d.token)` then `onLogin({ username, role })`.

## 7. frontend/src/components/AdminPanel.jsx (new)

Shows a create-user form (username, password, role dropdown) and a table of all users. Admins can change any other user's role via a dropdown and delete users. Guard: cannot change or delete own account.

## 8. frontend/src/App.jsx — auth gate

```jsx
const [user, setUser] = useState(null)
const [authChecked, setAuthChecked] = useState(false)

// On mount: validate stored token
useEffect(() => {
  if (!getToken()) { setAuthChecked(true); return }
  apiFetch('GET', '/api/auth/me').then(async r => {
    if (r.ok) setUser(await r.json())
    else setToken(null)
    setAuthChecked(true)
  }).catch(() => setAuthChecked(true))
}, [])

// Handle token expiry
useEffect(() => {
  const handler = () => setUser(null)
  window.addEventListener('auth:logout', handler)
  return () => window.removeEventListener('auth:logout', handler)
}, [])

if (!authChecked) return null           // avoid flash
if (!user) return <LoginPage onLogin={setUser} />
```

Add logout button + username display to the topbar.
Add `Admin` tab for users where `user.role === 'admin'`.

## 9. .env additions

```
JWT_SECRET=<32+ random hex chars>
JWT_EXPIRE_HOURS=24
FIRST_ADMIN_USERNAME=<your username>
FIRST_ADMIN_PASSWORD=<your password>
```

## 10. Verify

- [ ] `GET /api/wikis` without token → 401
- [ ] `POST /api/auth/login` with correct creds → token returned
- [ ] `GET /api/wikis` with `Authorization: Bearer <token>` → 200
- [ ] `GET /api/admin/users` with non-admin token → 403
- [ ] Self-registration → can log in immediately
- [ ] Admin creates user with admin role → new user can access admin panel
- [ ] Token expiry → frontend clears token and shows login page

## Known limitation

EventSource (SSE) does not support custom headers, so the stream endpoint accepts the JWT via `?token=` query param. This means the token appears in server access logs for stream requests. For higher security, replace `EventSource` with `fetch` + `ReadableStream` and send `Authorization: Bearer` header instead.

## Rules

- `GET /health` stays public — no auth dependency
- `POST /api/auth/register` and `POST /api/auth/login` are public
- All other endpoints require `Depends(get_current_user)` or `Depends(require_admin)`
- Never store the raw password — only the bcrypt hash
- Admin cannot demote or delete their own account
- `decode_token` raises `ValueError` on any JWTError — catch it in `get_current_user`
- `apiFetch` auto-detects `FormData` bodies and omits `Content-Type` (lets browser set multipart boundary)
- Dispatch `auth:logout` event on 401 so `App.jsx` can return to the login screen without a page reload
