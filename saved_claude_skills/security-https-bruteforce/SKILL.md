---
name: security-https-bruteforce
description: Checks and fixes HTTPS (self-signed TLS cert) and brute-force protection (rate limiting + server-side math CAPTCHA) for FastAPI + vanilla JS apps
trigger: /security-https-bruteforce
---

# /security-https-bruteforce

Audit and fix two vulnerabilities in a local FastAPI + vanilla JS credential manager:
1. **HTTPS** — app served over plain HTTP, exposing credentials and session tokens in transit
2. **Brute force** — unlock endpoint has no rate limiting or challenge, allowing unlimited password attempts

## What to do

### 1. Audit current state

Check whether each fix is already present before making changes:

- **HTTPS**: Does `scripts/run.sh` pass `--ssl-keyfile` and `--ssl-certfile` to uvicorn? Does `scripts/generate_cert.py` exist?
- **Rate limiting**: Is `slowapi` in `requirements.txt`? Does `backend/main.py` import and apply `@limiter.limit("5/minute")` on the unlock endpoint?
- **CAPTCHA**: Does `backend/captcha.py` exist with a `generate()` and `verify()` function? Does the unlock endpoint call `captcha.verify()`? Does the frontend fetch and display a CAPTCHA challenge?

Report what is already fixed and what still needs work before making any changes.

### 2. Fix HTTPS (if not already done)

**a. Create `scripts/generate_cert.py`** — generates a self-signed P-256 TLS certificate for localhost:

```python
#!/usr/bin/env python3
"""Generate a self-signed TLS cert for localhost if missing or expiring within 30 days."""
import datetime, ipaddress, sys
from pathlib import Path
from cryptography import x509
from cryptography.x509.oid import NameOID
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec

CERT_DIR = Path(__file__).parent.parent / "certs"
KEY_FILE = CERT_DIR / "server.key"
CERT_FILE = CERT_DIR / "server.crt"

def needs_regen():
    if not KEY_FILE.exists() or not CERT_FILE.exists():
        return True
    cert = x509.load_pem_x509_certificate(CERT_FILE.read_bytes())
    remaining = cert.not_valid_after_utc - datetime.datetime.now(datetime.timezone.utc)
    return remaining.days < 30

def generate():
    CERT_DIR.mkdir(exist_ok=True)
    key = ec.generate_private_key(ec.SECP256R1())
    KEY_FILE.write_bytes(key.private_bytes(
        serialization.Encoding.PEM,
        serialization.PrivateFormat.TraditionalOpenSSL,
        serialization.NoEncryption(),
    ))
    subject = issuer = x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, "localhost")])
    cert = (
        x509.CertificateBuilder()
        .subject_name(subject)
        .issuer_name(issuer)
        .public_key(key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(datetime.datetime.now(datetime.timezone.utc))
        .not_valid_after(datetime.datetime.now(datetime.timezone.utc) + datetime.timedelta(days=825))
        .add_extension(x509.SubjectAlternativeName([
            x509.DNSName("localhost"),
            x509.IPAddress(ipaddress.IPv4Address("127.0.0.1")),
        ]), critical=False)
        .sign(key, hashes.SHA256())
    )
    CERT_FILE.write_bytes(cert.public_bytes(serialization.Encoding.PEM))
    print("TLS certificate generated.")

if needs_regen():
    generate()
else:
    print("TLS certificate is valid.")
```

**b. Update `scripts/run.sh`** to call `generate_cert.py` and pass SSL flags to uvicorn:

```bash
#!/usr/bin/env bash
set -e
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
cd "$PROJECT_ROOT"

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt --quiet

python3 scripts/generate_cert.py

python3 -m uvicorn backend.main:app \
  --host 127.0.0.1 --port 8000 \
  --ssl-keyfile certs/server.key \
  --ssl-certfile certs/server.crt \
  --reload
```

**c. Update CORS** in `backend/main.py` to use `https://`:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://localhost:8000", "https://127.0.0.1:8000"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**d. Update `.gitignore`** to exclude cert files:

```
certs/server.key
certs/server.crt
```

**e. Tell the user**: "Access the app at `https://localhost:8000`. Your browser will show a security warning for the self-signed cert — click Advanced → Proceed. This is expected for localhost-only apps."

### 3. Fix brute force (if not already done)

**a. Add `slowapi` to `requirements.txt`**:

```
slowapi==0.1.9
```

**b. Create `backend/captcha.py`** — server-side one-time math CAPTCHA with TTL:

```python
import secrets, time
from typing import Any

_store: dict[str, dict[str, Any]] = {}
_TTL = 300  # 5 minutes

def generate() -> dict:
    """Return {id, question}. Answer is never sent to the client."""
    _purge()
    a, b = secrets.randbelow(15) + 1, secrets.randbelow(15) + 1
    cid = secrets.token_urlsafe(16)
    _store[cid] = {"answer": a + b, "expires": time.time() + _TTL}
    return {"id": cid, "question": f"{a} + {b} = ?"}

def verify(cid: str, answer: int) -> bool:
    """One-time-use: entry is deleted whether correct or not."""
    entry = _store.pop(cid, None)
    if not entry or time.time() > entry["expires"]:
        return False
    return entry["answer"] == answer

def _purge():
    now = time.time()
    expired = [k for k, v in _store.items() if now > v["expires"]]
    for k in expired:
        del _store[k]
```

**c. Update `backend/schemas.py`** — add `captcha_id` and `captcha_answer` to `UnlockRequest`:

```python
class UnlockRequest(BaseModel):
    master_password: str
    captcha_id: str
    captcha_answer: int
```

**d. Update `backend/main.py`** — add rate limiter, CAPTCHA endpoint, and protect unlock:

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address
from backend import captcha

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/api/captcha")
def get_captcha():
    return captcha.generate()

@app.post("/api/vault/unlock", response_model=SessionResponse)
@limiter.limit("5/minute")
def vault_unlock(request: Request, body: UnlockRequest, db: Session = Depends(get_db)):
    if not captcha.verify(body.captcha_id, body.captcha_answer):
        raise HTTPException(status_code=422, detail="Invalid or expired security challenge")
    # ... rest of unlock logic unchanged
```

**e. Update `frontend/index.html`** — add CAPTCHA UI to the unlock form:

```html
<div class="mb-3 captcha-box" id="captchaBox">
  <label class="form-label">Security Challenge</label>
  <div class="d-flex align-items-center gap-2 mb-2">
    <span id="captchaQuestion" class="captcha-question">Loading...</span>
    <button type="button" class="btn btn-sm btn-outline-secondary" onclick="refreshCaptcha()">↺</button>
  </div>
  <input type="number" class="form-control" id="captchaAnswer" placeholder="Answer" required />
</div>
```

**f. Update `frontend/app.js`** — fetch CAPTCHA on load, include in unlock request:

```js
let captchaId = null;

async function fetchCaptcha() {
  const data = await api('GET', '/api/captcha');
  captchaId = data.id;
  document.getElementById('captchaQuestion').textContent = data.question;
  document.getElementById('captchaAnswer').value = '';
}

async function refreshCaptcha() { await fetchCaptcha(); }

// Call fetchCaptcha() when rendering the unlock screen
// In unlock submit handler, include captcha fields:
const body = {
  master_password: password,
  captcha_id: captchaId,
  captcha_answer: parseInt(document.getElementById('captchaAnswer').value, 10),
};
```

### 4. Verify

- Restart the server with `./scripts/run.sh`
- Confirm the app loads at `https://localhost:8000` (not http)
- Confirm the unlock form shows a math question
- Confirm submitting a wrong answer shows an error without consuming a rate-limit slot
- Confirm 5 failed unlock attempts in 1 minute triggers a 429 response

## Rules

- Never store the CAPTCHA answer in the HTTP response — only `id` and `question` go to the client
- CAPTCHA entries must be one-time-use: delete from `_store` on any `verify()` call, success or failure
- Rate limit applies to the `/api/vault/unlock` endpoint only — do not rate-limit the `/api/captcha` endpoint
- The cert must include both `localhost` DNS SAN and `127.0.0.1` IP SAN or some browsers will reject it
- Never commit `certs/server.key` or `certs/server.crt` to git
- CORS `allow_origins` must use `https://` after enabling TLS
- If the project already uses a venv, activate it before running `generate_cert.py`
