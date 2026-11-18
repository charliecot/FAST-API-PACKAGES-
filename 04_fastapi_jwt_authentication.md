# Authentication in FastAPI: JWT, Login, Logout, Passwords, Refresh Tokens, and Alternatives

## 1. Authentication vs authorisation

**Authentication** asks: who are you?

**Authorisation** asks: what are you allowed to do?

A typical API workflow:

1. User registers or already has an account.
2. User submits credentials to a login endpoint.
3. Server verifies the password hash.
4. Server issues an access token.
5. Client sends the token with later requests.
6. Server validates the token and identifies the user.
7. Permission logic decides whether that user may perform the action.

JWT is a token format. It is not, by itself, a complete authentication system. You still need password hashing, user storage, token validation, expiration, refresh strategy, logout/revocation policy, and permissions.

## 2. JWT structure

A JSON Web Token commonly has three Base64URL-encoded sections:

```text
header.payload.signature
```

- Header: token type and signing algorithm.
- Payload: claims such as subject (`sub`), expiry (`exp`), issued-at (`iat`), issuer (`iss`), audience (`aud`).
- Signature: protects integrity and authenticity.

A signed JWT is generally **not encrypted**. Anyone who obtains it can usually decode its header and payload. Never put passwords, private secrets, or sensitive personal information in its payload.

The signature helps detect tampering; it does not hide the payload.

## 3. Install libraries

A common explicit approach:

```bash
uv add fastapi uvicorn sqlmodel "pwdlib[argon2]" pyjwt
```

For password hashing, FastAPI's current security tutorial uses `pwdlib` with Argon2. Another established option is Passlib, although compatibility and maintenance should be checked for the versions you choose.

JWT library options:
- **PyJWT**: focused JWT encode/decode library; relatively small and direct.
- **python-jose**: another JWT/JWS/JWE-related library used in many FastAPI examples; check maintenance and dependency status before choosing.
- **Authlib**: broader OAuth/OIDC and JOSE functionality.
- **FastAPI Users**: higher-level user-management package with authentication strategies and integrations; more opinionated.
- **fastapi-users** and other frameworks can reduce boilerplate, but you should understand the underlying security concepts.

There is no need to use a large framework just to sign and verify a JWT. PyJWT plus a password-hashing library is a straightforward low-level option.

## 4. User model

```python
from sqlmodel import SQLModel, Field

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    email: str = Field(index=True, unique=True)
    hashed_password: str
    is_active: bool = True
    is_admin: bool = False
```

Never store a plaintext password. Store a password hash produced by a password-hashing algorithm.

In a real application, also consider email verification, timestamps, account lockout/rate limits, password reset state, and audit/security events.

## 5. Password hashing

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()

def hash_password(password: str) -> str:
    return password_hash.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return password_hash.verify(plain_password, hashed_password)
```

Hashing is one-way: the server verifies a submitted password against the stored hash. It does not decrypt the stored hash.

Why Argon2? It is a modern password-hashing algorithm designed to make large-scale guessing expensive. Do not use ordinary SHA-256 or MD5 directly for password storage; they are not password-hashing algorithms.

Also:
- enforce reasonable password length limits;
- use HTTPS;
- rate-limit login and reset endpoints;
- avoid revealing whether an email exists during password-reset requests;
- never log passwords or tokens.

## 6. Configuration and secret key

Generate a strong random secret:

```bash
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

Store it in an environment variable, not in Git:

```env
JWT_SECRET_KEY=replace-with-a-long-random-secret
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
```

Load configuration using `pydantic-settings`:

```bash
uv add pydantic-settings
```

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 15

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

settings = Settings()
```

Use separate secrets for development, testing, and production. For asymmetric algorithms such as RS256, the signing private key and verification public key have different roles. Do not casually change algorithms without understanding key management.

## 7. Create an access token with PyJWT

```python
from datetime import datetime, timedelta, timezone
import jwt

def create_access_token(subject: str) -> str:
    now = datetime.now(timezone.utc)

    payload = {
        "sub": subject,
        "iat": now,
        "exp": now + timedelta(minutes=15),
        "type": "access",
    }

    return jwt.encode(
        payload,
        settings.jwt_secret_key,
        algorithm=settings.jwt_algorithm,
    )
```

`sub` identifies the token subject, often the user's stable ID. Use a string representation. `exp` is the expiration time. The `type` claim is an application convention that helps distinguish access and refresh tokens.

## 8. OAuth2 password form and login endpoint

FastAPI provides `OAuth2PasswordBearer` to extract a bearer token from the Authorization header. For a classic OAuth2 password form example, `OAuth2PasswordRequestForm` parses form-encoded username/password fields. The OAuth2 password grant is no longer recommended for new OAuth deployments; for first-party learning or a simple legacy-style API, understand the flow, but prefer Authorization Code with PKCE or a proper OIDC provider for modern third-party login.

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlmodel import Session, select

@app.post("/auth/login")
def login(
    form: OAuth2PasswordRequestForm = Depends(),
    session: Session = Depends(get_session),
):
    # This example uses email in the form's "username" field.
    user = session.exec(
        select(User).where(User.email == form.username)
    ).first()

    if user is None or not verify_password(
        form.password, user.hashed_password
    ):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if not user.is_active:
        raise HTTPException(status_code=400, detail="Inactive account")

    token = create_access_token(str(user.id))

    return {
        "access_token": token,
        "token_type": "bearer",
    }
```

The OAuth2 password form sends `application/x-www-form-urlencoded`, not JSON. If your frontend wants JSON login, define a Pydantic request schema and handle it explicitly. Do not confuse the two request formats.

## 9. Decode and validate a token

```python
from jwt import InvalidTokenError

def decode_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            settings.jwt_secret_key,
            algorithms=[settings.jwt_algorithm],
            options={"require": ["exp", "sub"]},
        )
    except InvalidTokenError:
        raise HTTPException(
            status_code=401,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="Invalid token type")

    return payload
```

Always pass an explicit allowlist of accepted algorithms. Do not trust an algorithm chosen by an untrusted token header. In production, validate issuer and audience when your architecture uses them.

## 10. Get the current user with a dependency

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: Session = Depends(get_session),
):
    payload = decode_access_token(token)
    user_id = payload["sub"]

    user = session.get(User, int(user_id))

    if user is None or not user.is_active:
        raise HTTPException(status_code=401, detail="User not available")

    return user
```

Use it in endpoints:

```python
@app.get("/auth/me")
def read_me(current_user: User = Depends(get_current_user)):
    return {
        "id": current_user.id,
        "email": current_user.email,
    }
```

The dependency extracts the bearer token, verifies it, loads the user, and provides the user object to the endpoint.

## 11. Logout: why JWT logout is different

A self-contained access JWT remains cryptographically valid until it expires unless the server adds a revocation mechanism.

Deleting the token from frontend memory is a client-side logout, but a stolen copy may still work.

Common strategies:

### Strategy A — short-lived access token only

- Access token expires quickly.
- Client discards it on logout.
- No server-side revocation list.
- Simple, but stolen tokens remain usable until expiry.

### Strategy B — refresh token plus access token

- Short-lived access token is sent to API endpoints.
- Longer-lived refresh token obtains a new access token.
- Refresh token is stored/managed securely.
- Logout revokes the refresh token or token family.
- Access token may remain valid until its short expiry unless separately revoked.

### Strategy C — token denylist / revocation table

Store a token identifier (`jti`) or token hash and expiration in a database/Redis denylist. During authentication, reject revoked tokens. Keep entries only until the token would naturally expire.

Do not store raw bearer tokens in logs. A `jti` is a token identifier, not a replacement for secure storage.

### Example conceptual logout

```python
@app.post("/auth/logout", status_code=204)
def logout(current_user=Depends(get_current_user)):
    # With stateless access-only JWTs, there is no server-side token
    # to invalidate here. The client must discard its token.
    # If using refresh tokens, revoke the refresh-token record here.
    return None
```

Be honest in your API design: this endpoint alone does not invalidate an already-issued stateless access JWT.

## 12. Refresh token design

A robust design usually has:
- short-lived access JWT;
- refresh token with longer expiry;
- refresh token stored as a hashed value in the database;
- rotation on each refresh;
- detection/revocation of reused refresh tokens;
- logout that revokes the refresh token;
- secure cookie or carefully protected client storage.

A refresh record might store:

```text
id | user_id | token_hash | expires_at | revoked_at | created_at
```

On refresh:
1. Receive refresh token.
2. Hash it and find the matching stored record.
3. Check not expired/revoked.
4. Rotate: revoke old record and create a new refresh token.
5. Issue a new access token.
6. If an old rotated token is reused, consider revoking the token family and requiring reauthentication.

Do not treat a long-lived access token as a refresh-token design.

## 13. Password change

A logged-in user changing their password should normally prove knowledge of the current password.

```python
class PasswordChange(SQLModel):
    current_password: str
    new_password: str

@app.post("/auth/change-password", status_code=204)
def change_password(
    payload: PasswordChange,
    user: User = Depends(get_current_user),
    session: Session = Depends(get_session),
):
    if not verify_password(payload.current_password, user.hashed_password):
        raise HTTPException(status_code=400, detail="Current password is incorrect")

    user.hashed_password = hash_password(payload.new_password)
    session.add(user)
    session.commit()

    # Consider revoking all refresh tokens/sessions after a password change.
    return None
```

Add password policy and length validation. Avoid overly complex arbitrary rules; use a sensible minimum length and block known-compromised/common passwords where possible.

## 14. Password reset by email

A secure reset flow is different from a logged-in password change.

1. Client submits email to `/auth/forgot-password`.
2. Always return a generic response such as "If the account exists, instructions have been sent."
3. Generate a cryptographically random, single-use reset token.
4. Store only a hash of the reset token, with user ID and short expiration.
5. Email the raw token in a HTTPS reset link.
6. User submits token and new password to `/auth/reset-password`.
7. Hash and compare token, verify unused and unexpired, update password.
8. Mark token used; revoke existing sessions/refresh tokens as policy requires.

Do not expose whether an email address exists. Do not log reset tokens. Apply rate limiting. A signed JWT can be used for reset tokens, but a random opaque one-time token stored hashed in the database is often easier to revoke and reason about.

## 15. Registration

```python
class UserCreate(SQLModel):
    email: str
    password: str

@app.post("/auth/register", status_code=201)
def register(payload: UserCreate, session: Session = Depends(get_session)):
    existing = session.exec(
        select(User).where(User.email == payload.email)
    ).first()

    if existing:
        raise HTTPException(status_code=409, detail="Account already exists")

    user = User(
        email=payload.email,
        hashed_password=hash_password(payload.password),
    )
    session.add(user)
    session.commit()
    session.refresh(user)

    return {"id": user.id, "email": user.email}
```

In a production application, enforce uniqueness at the database level and handle an `IntegrityError` race condition. Validate and normalize email consistently. Do not return the hashed password.

## 16. Social login: OAuth2 and OpenID Connect

Social login is not simply "accept a Google username". The usual flow is:
- redirect user to identity provider;
- user authenticates there;
- provider redirects to registered callback;
- backend validates state and exchanges authorization code;
- backend validates ID token/identity using OIDC rules;
- backend maps provider identity to a local user;
- application creates its own session or token.

Use a maintained OAuth/OIDC library or identity provider integration. Do not trust profile data from an unverified browser request.

## 17. Token storage choices

### Browser localStorage
Easy to use, but JavaScript can read it. An XSS vulnerability can steal tokens.

### HttpOnly Secure cookie
JavaScript cannot read it, which reduces token theft through simple XSS. Cookies are automatically attached by the browser, so CSRF protections and SameSite configuration matter.

### In-memory access token
Reduces persistence but disappears on page reload. Often combined with a refresh cookie.

There is no universal storage answer. Choose based on frontend architecture and threat model. Always use HTTPS in production.

## 18. Security checklist

- [ ] Hash passwords with Argon2id or another appropriate password hash.
- [ ] Never store plaintext passwords.
- [ ] Use strong secrets from environment/secret manager.
- [ ] Keep access tokens short-lived.
- [ ] Validate `exp`, `sub`, token type, and relevant issuer/audience.
- [ ] Explicitly allow accepted JWT algorithms.
- [ ] Do not put secrets in JWT payloads.
- [ ] Use refresh-token rotation/revocation if refresh tokens exist.
- [ ] Understand that stateless JWT logout does not instantly revoke access tokens.
- [ ] Rate-limit login and reset flows.
- [ ] Use generic forgot-password responses.
- [ ] Use HTTPS.
- [ ] Protect cookie-based flows against CSRF.
- [ ] Revoke sessions after password reset/change when appropriate.
- [ ] Test authentication and authorisation separately.

## 19. Official references

- FastAPI security tutorial: https://fastapi.tiangolo.com/tutorial/security/
- PyJWT: https://pyjwt.readthedocs.io/
- pwdlib: https://github.com/frankie567/pwdlib
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP JSON Web Token Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- OAuth 2.0 Security Best Current Practice (RFC 9700): https://www.rfc-editor.org/rfc/rfc9700
