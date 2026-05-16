---
name: security-web-vulns
description: Checks and fixes 5 common web vulnerabilities in FastAPI + vanilla JS apps — session token leakage, stored XSS, javascript: URL injection, mass-assignment in PUT, and secrets in DOM attributes
trigger: /security-web-vulns
---

# /security-web-vulns

Audit and fix five vulnerabilities in a FastAPI + vanilla JS credential manager:

3. **Session token in server logs** — token passed as URL query param is recorded in access logs
4. **Stored XSS** — user-supplied strings rendered via `innerHTML` without escaping
5. **`javascript:` URL injection** — attacker-controlled URL fields can run JS on click
6. **Mass-assignment in PUT** — `PUT /api/credentials/{id}` accepts raw `dict` body and may set encrypted columns directly
7. **Secrets in DOM attributes** — secret values stored in `data-secret` attributes are readable via DevTools even when "masked"

## What to do

### 1. Audit current state

Check whether each fix is already present before making changes:

- **#3**: Does `backend/deps.py` use only `Header(...)` to read the session token (not a `Query()` param)?
- **#4**: Does `frontend/app.js` have an `esc()` function that HTML-encodes user strings, and are all user-supplied strings passed through it before being included in template literals assigned to `innerHTML`?
- **#5**: Does `frontend/app.js` validate URL fields against an `http://` / `https://` allowlist before rendering as `<a href>`?
- **#6**: Does `backend/crud.py`'s `update_credential` have an explicit `_PLAINTEXT_FIELDS` allowlist and map each sensitive field name (e.g. `"token"`, `"password"`) to its encrypted column? Are `*_enc` column names and `id`/`cred_type`/`created_at` silently ignored?
- **#7**: Do secret fields in the credential detail view use `data-field="fieldName"` (a key into `state.viewingCredential`) instead of `data-secret="actualValue"`? Do `toggleSecret()` and `copyField()` read from JS state rather than the DOM?

Report what is already fixed and what still needs work before making any changes.

### 2. Fix #3 — Session token in server logs

**Remove any `Query()` session token parameter** from `backend/deps.py`. The token must only be read from the `X-Session-Token` request header:

```python
# backend/deps.py
from fastapi import Header, HTTPException
from cryptography.fernet import Fernet
from backend.session import vault_session

async def require_unlocked_session(x_session_token: str = Header(...)) -> Fernet:
    if not vault_session.verify_token(x_session_token):
        raise HTTPException(status_code=401, detail="Invalid or expired session token")
    vault_session.reset_timer()
    return vault_session.get_fernet()
```

**Remove any `?token=` query string usage** from `frontend/app.js`. Every API call must send the token as a header:

```js
async function api(method, path, body = null) {
  const headers = { 'Content-Type': 'application/json' };
  if (state.sessionToken) headers['X-Session-Token'] = state.sessionToken;
  const res = await fetch(path, {
    method,
    headers,
    body: body ? JSON.stringify(body) : null,
  });
  // ...
}
```

### 3. Fix #4 — Stored XSS via innerHTML

Add an `esc()` function that converts the five HTML special characters to entities:

```js
function esc(s) {
  return String(s ?? '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}
```

Wrap every user-supplied value in `esc()` before interpolating into template literals that are assigned to `innerHTML`. This includes: credential names, tags, notes, service names, hosts, public keys, and any other plaintext fields displayed in lists or detail views.

Do NOT call `esc()` on values rendered via `textContent` (that property already prevents HTML injection).

### 4. Fix #5 — `javascript:` URL injection

Before rendering a URL field as a clickable `<a href>`, validate it against an allowlist:

```js
function safeUrl(raw) {
  const s = (raw || '').trim();
  if (s.startsWith('http://') || s.startsWith('https://')) return s;
  return null;
}
```

In template code, use `safeUrl()` and fall back to plain text if the URL is not safe:

```js
const url = safeUrl(cred.url);
const urlHtml = url
  ? `<a href="${esc(url)}" target="_blank" rel="noopener noreferrer">${esc(url)}</a>`
  : esc(cred.url || '—');
```

Apply the same check to any SSH host field that gets rendered as a link, if applicable.

### 5. Fix #6 — Mass-assignment in PUT endpoint

In `backend/crud.py`, replace any `setattr(row, field, value)` fallback with an explicit field mapping. The update function must:

- Map friendly field names (e.g. `"token"`, `"password"`) to their encrypted columns via `_enc()`
- Allow updates to a fixed set of plaintext fields via `setattr`
- Silently ignore everything else — including `id`, `cred_type`, `created_at`, and any `*_enc` column names

```python
_PLAINTEXT_FIELDS = {"name", "tags", "service_name", "host", "public_key"}

def update_credential(db, f, row, updates):
    for field, value in updates.items():
        if value is None:
            continue
        if field == "notes":
            row.notes_enc = _enc(f, value)
        elif field == "token":
            row.token_enc = _enc(f, value)
        elif field == "url":
            row.url_enc = _enc(f, value)
        elif field == "username":
            row.username_enc = _enc(f, value)
        elif field == "password":
            row.password_enc = _enc(f, value)
        elif field == "private_key":
            row.private_key_enc = _enc(f, value)
        elif field == "passphrase":
            row.passphrase_enc = _enc(f, value)
        elif field == "pairs":
            row.pairs_enc = _enc(f, json.dumps(value))
        elif field in _PLAINTEXT_FIELDS:
            setattr(row, field, value)
        # all other keys (id, cred_type, *_enc columns, etc.) are silently ignored
    row.updated_at = datetime.utcnow()
    db.commit()
    db.refresh(row)
    return row
```

Also update the PUT route in `backend/main.py` to accept a typed Pydantic body instead of a raw `dict`, if the credential type is known at update time.

### 6. Fix #7 — Secrets in DOM attributes

Replace `data-secret="<actual_value>"` on secret field elements with `data-field="<fieldName>"`. Secret values live only in `state.viewingCredential` (a JS variable in memory), not in the DOM.

**`secretRow()` helper** — builds a masked row using a field key, not a value:

```js
function secretRow(label, fieldName, masked = true) {
  const value = state.viewingCredential ? (state.viewingCredential[fieldName] || '') : '';
  const display = masked ? '••••••••' : esc(value);
  const cls = masked ? 'masked' : '';
  return `<div class="detail-row">
    <div class="detail-label">${label}</div>
    <div class="secret-field">
      <div class="secret-value ${cls}" data-field="${fieldName}">${display}</div>
      <button class="btn-icon" onclick="toggleSecret(this)">👁</button>
      <button class="btn-icon" data-field="${fieldName}" onclick="copyField(this.dataset.field)">📋</button>
    </div>
  </div>`;
}
```

**State lookup helpers** — read values from `state.viewingCredential`, not the DOM:

```js
function getCredentialField(fieldPath) {
  if (!state.viewingCredential) return '';
  if (fieldPath.startsWith('pairs.'))
    return (state.viewingCredential.pairs || {})[fieldPath.slice(6)] || '';
  return state.viewingCredential[fieldPath] || '';
}

function copyField(fieldPath) { copyText(getCredentialField(fieldPath)); }

function toggleSecret(btn) {
  const el = btn.previousElementSibling;
  if (el.classList.contains('masked')) {
    el.textContent = getCredentialField(el.dataset.field);
    el.classList.remove('masked');
    btn.textContent = '🙈';
  } else {
    el.textContent = '••••••••';
    el.classList.add('masked');
    btn.textContent = '👁';
  }
}
```

For env var pairs, use `data-field="pairs.<KEY>"` so `getCredentialField` can resolve the nested value.

### 7. Verify

After applying fixes, check:

- [ ] `curl -v https://localhost:8000/api/vault/status` — no `?token=` in the request URL visible in server logs
- [ ] Create a credential with `<img src=x onerror=alert(1)>` as the name — confirm it displays as literal text, not an alert
- [ ] Create a credential with `javascript:alert(1)` as the URL — confirm it is not rendered as a clickable link
- [ ] Send `PUT /api/credentials/1` with body `{"cred_type": "api_key", "token_enc": "EVIL"}` — confirm the encrypted column is not overwritten
- [ ] Open DevTools → Elements, view a credential with a secret — confirm no `data-secret` attribute contains the actual value

## Rules

- `esc()` must encode all five HTML special characters: `& < > " '`
- Call `esc()` on every user-supplied string inserted into `innerHTML` — no exceptions
- URL allowlist must use `startsWith`, not a regex, to avoid bypass via whitespace or unicode
- `data-field` must store a lookup key (field name), never the secret value itself
- `toggleSecret()` must read from `state.viewingCredential`, not `el.dataset`
- The update allowlist in `crud.py` must explicitly enumerate every allowed field; no catch-all `setattr` fallback for unknown fields
- Do not change the API contract (endpoint paths, response shapes) — only fix the internal handling
