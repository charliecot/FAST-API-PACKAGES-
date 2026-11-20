# FastAPI Authentication – Social Login with OAuth 2.0 and OpenID Connect

> A practical guide to adding Google, GitHub, Facebook, Microsoft, and other identity providers to FastAPI. It explains the protocols, libraries, callback flow, account linking, security, and implementation choices.

## 1. What is social authentication?

Social authentication lets a user sign in to your application using an identity account they already have, such as Google or GitHub.

Instead of asking your application to verify a new password, the user authenticates with the provider. Your application receives a verified identity and creates or finds a local user account.

Example:

```text
User -> React frontend -> FastAPI
                    |
                    v
              Google sign-in
                    |
             User authenticates
                    |
                    v
            Google redirects back
                    |
                    v
            FastAPI callback
                    |
             Verify identity
                    |
                    v
       Find/create local application user
                    |
                    v
       Issue YOUR application's JWT/session
```

The provider proves who the user is. Your application still controls its own users, roles, permissions, orders, and data.

## 2. OAuth 2.0 versus OpenID Connect

These terms are related but not identical.

### OAuth 2.0

OAuth 2.0 is primarily an **authorization framework**. It lets an application obtain limited access to a resource with the user's authorization.

Example: allowing your app to access a user's GitHub profile or Google Calendar.

OAuth access tokens are not automatically proof of the user's identity.

### OpenID Connect (OIDC)

OpenID Connect is an identity layer built on OAuth 2.0. It defines how a client can authenticate a user and receive identity claims, commonly in an ID token.

For login, prefer a provider's OpenID Connect implementation where available.

Important:
- An **ID token** communicates authentication/identity claims to the OIDC client.
- An **access token** authorizes access to a provider's API.
- A **refresh token** may be used to obtain new provider access tokens when allowed.
- Your application's own JWT is a separate credential for your API.

Do not confuse the provider's token with the token your FastAPI application issues.

## 3. Four popular providers (plus more)

<text>Provider availability and scopes can change. Always check each provider's current developer documentation before implementation.</text>

### 1. Google

<AsyncImage query="Google sign in logo official" aspectRatio="16:9" fit="contain" maxHeight={90}/>
- Common protocol: OpenID Connect over OAuth 2.0.
- Common identity scopes: `openid`, `email`, `profile`.
- Good for consumer applications because many users have Google accounts.
- Google identity claims commonly include a stable subject identifier (`sub`), email, and profile information, depending on scopes and consent.

Developer console: https://console.cloud.google.com/

### 2. GitHub

<AsyncImage query="GitHub logo" aspectRatio="16:9" fit="contain" maxHeight={90}/>
- Common protocol: OAuth 2.0.
- Useful for developer-focused applications.
- User email may require an additional API request and appropriate permission; do not assume the profile endpoint always returns a public email.
- GitHub OAuth login is not the same as a general OIDC flow in every integration. Follow GitHub's current OAuth App or GitHub App documentation.

Developer settings: https://github.com/settings/developers

### 3. Facebook

<AsyncImage query="Meta Facebook logo" aspectRatio="16:9" fit="contain" maxHeight={90}/>
- Uses Meta's OAuth-based Login product.
- Useful when your audience uses Facebook.
- Access to profile/email fields is controlled by permissions, app review, and platform policies.
- App setup and review requirements may apply.

Developer portal: https://developers.facebook.com/

### 4. Microsoft

<AsyncImage query="Microsoft logo" aspectRatio="16:9" fit="contain" maxHeight={90}/>
- Supports OpenID Connect and OAuth 2.0.
- Can support personal Microsoft accounts and organizational/work accounts depending on app registration configuration.
- Useful for business applications and Microsoft 365/Azure ecosystems.
- Tenant configuration affects who can sign in.

Developer portal: https://entra.microsoft.com/

### Other possible providers

- Apple Sign in: useful for iOS-oriented consumer applications.
- Discord: useful for community and gaming applications.
- LinkedIn: useful for professional identity use cases.
- GitLab: useful for developer platforms.
- Amazon: useful for applications targeting Amazon account holders.

Choose providers based on your actual users. You do not need to implement every provider.

## 4. Libraries for FastAPI social login

FastAPI does not ship with one built-in social-auth system equivalent to Django Allauth. You choose a library or implement the protocol using a maintained OAuth/OIDC client.

### Option A: Authlib

Authlib is a widely used Python library for OAuth 1.0, OAuth 2.0, and OpenID Connect integrations.

Good when:
- You want to integrate providers directly in FastAPI/Starlette.
- You need control over routes and callbacks.
- You are comfortable understanding the OAuth/OIDC flow.

Install:

```bash
uv add authlib httpx itsdangerous
```

`itsdangerous` is commonly used by Starlette's SessionMiddleware for signed session cookies. Review session-cookie configuration carefully.

### Option B: fastapi-users

`fastapi-users` provides reusable user-management and authentication components, and has offered OAuth support through its OAuth integration.

Good when:
- You want a framework-style user management solution.
- You want to avoid implementing every user-management feature yourself.

Check the current project's maintenance status, supported FastAPI/SQLAlchemy versions, provider support, and integration documentation before selecting it. Its integration model and APIs may vary by release.

### Option C: Auth0 / Clerk / WorkOS / Keycloak

These are identity platforms or identity servers rather than just Python libraries.

- **Auth0:** managed identity platform.
- **Clerk:** managed authentication/user management, often used with modern frontend applications.
- **WorkOS:** identity features often aimed at business/SaaS applications.
- **Keycloak:** self-hostable identity and access management server.

Your FastAPI app validates tokens or integrates with the provider's supported protocol. These services can reduce the amount of identity infrastructure you must build, but introduce provider configuration, pricing, and operational considerations.

### Option D: Direct OIDC client

For an OIDC provider, use a maintained OIDC-capable library such as Authlib or a suitable standards-compliant client. Do not hand-roll signature verification, issuer checks, key rotation, nonce handling, or token validation.

## 5. Provider setup: the general process

For each provider:

1. Create a developer application.
2. Configure the application's website/homepage.
3. Register the exact callback/redirect URI.
4. Select required scopes/permissions.
5. Obtain a client ID.
6. Obtain a client secret (for confidential server-side clients).
7. Store credentials securely in environment variables.
8. Configure development and production callback URLs separately.
9. Test consent, cancellation, errors, and successful login.

Example callback:

```text
http://localhost:8000/auth/google/callback
```

Production example:

```text
https://api.example.com/auth/google/callback
```

The redirect URI used in the authorization request must match the registered URI exactly, according to the provider's rules.

Never put a client secret in a React/Vite frontend environment variable. Frontend variables are visible to users. A confidential client secret belongs on the server.

## 6. Understand the authorization-code flow

For a server-side FastAPI application, the Authorization Code flow is a common choice. Use PKCE where supported and appropriate; it is especially important for public clients and is also a strong modern practice for many authorization-code integrations.

High-level flow:

1. User clicks “Continue with Google”.
2. FastAPI creates a random `state` value and, for OIDC, a `nonce`.
3. FastAPI redirects the browser to the provider's authorization endpoint.
4. User authenticates and consents.
5. Provider redirects to your callback with an authorization code and state.
6. FastAPI checks state.
7. FastAPI exchanges the code for tokens over a back channel.
8. FastAPI validates the ID token (issuer, audience, signature, expiry, nonce, and relevant claims).
9. FastAPI finds or creates a local user.
10. FastAPI issues its own application session or JWT.

The authorization code is short-lived and should be exchanged server-side. Do not treat the callback's query parameters as verified identity.

## 7. Example Authlib integration with Starlette/FastAPI

This is a learning skeleton. You must configure real provider metadata, client credentials, session security, local-user persistence, and application token issuance.

### Environment variables

```dotenv
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
SESSION_SECRET=long-random-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/auth/google/callback
```

### Configure the app

```python
import os

from authlib.integrations.starlette_client import OAuth
from fastapi import FastAPI, Request
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()

app.add_middleware(
    SessionMiddleware,
    secret_key=os.environ["SESSION_SECRET"],
    same_site="lax",
    https_only=False,  # Local development only. Set True in HTTPS production.
)

oauth = OAuth()

oauth.register(
    name="google",
    client_id=os.environ["GOOGLE_CLIENT_ID"],
    client_secret=os.environ["GOOGLE_CLIENT_SECRET"],
    server_metadata_url=(
        "https://accounts.google.com/.well-known/openid-configuration"
    ),
    client_kwargs={
        "scope": "openid email profile",
    },
)
```

The OIDC discovery document supplies endpoints and signing-key metadata. Use the library's supported OIDC validation functionality rather than decoding the JWT without verification.

### Start login

```python
from fastapi.responses import RedirectResponse

@app.get("/auth/google/login")
async def google_login(request: Request):
    redirect_uri = os.environ["GOOGLE_REDIRECT_URI"]
    return await oauth.google.authorize_redirect(
        request,
        redirect_uri,
    )
```

Authlib manages protocol details such as state through its integration. Do not remove state validation.

### Callback

```python
@app.get("/auth/google/callback")
async def google_callback(request: Request):
    token = await oauth.google.authorize_access_token(request)

    # For OIDC, Authlib integrations can expose parsed/validated user
    # information depending on provider metadata and library version.
    user_info = token.get("userinfo")

    if not user_info:
        # Handle missing/invalid identity information safely.
        raise HTTPException(
            status_code=400,
            detail="Unable to obtain verified user information",
        )

    provider_subject = user_info["sub"]
    email = user_info.get("email")
    email_verified = user_info.get("email_verified", False)

    # 1. Find/create your local user using provider + provider_subject.
    # 2. Apply your account-linking policy.
    # 3. Issue YOUR application's access/refresh tokens or session.
    # 4. Redirect to your frontend or return a safe response.

    return {
        "provider": "google",
        "provider_subject": provider_subject,
        "email": email,
        "email_verified": email_verified,
    }
```

This deliberately leaves local user persistence and token creation as explicit application steps. Do not ship a callback that returns provider tokens or sensitive claims to the browser without a design reason.

Authlib's exact response parsing behavior can depend on version and provider configuration. Follow the installed version's documentation and test issuer, audience, signature, expiry, nonce, and state validation.

## 8. Your local user model

Do not create a separate application user table for every provider. A common design uses a local `User` table and a related `OAuthAccount` table.

Example SQLModel design:

```python
from sqlmodel import SQLModel, Field, Relationship
from typing import Optional

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    email: str | None = Field(default=None, index=True)
    full_name: str | None = None
    is_active: bool = True

    oauth_accounts: list["OAuthAccount"] = Relationship(
        back_populates="user"
    )


class OAuthAccount(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)

    # Example: "google", "github", "facebook", "microsoft"
    provider: str = Field(index=True)

    # Stable unique subject/user identifier from that provider.
    provider_user_id: str = Field(index=True)

    user_id: int = Field(foreign_key="user.id")
    user: Optional[User] = Relationship(back_populates="oauth_accounts")
```

Add a **database-level unique constraint** over `(provider, provider_user_id)`. In SQLModel this can be defined with SQLAlchemy table arguments. A simple pair of indexed columns alone does not guarantee uniqueness.

Why store provider identity separately?

- A user can connect Google and GitHub to one local account.
- Provider email addresses can change.
- Email may be missing or unverified.
- Provider subject IDs are the stable identity key within that provider.

## 9. Account creation and account linking

A safe account lookup strategy is:

```text
(provider, provider_subject)
        |
        +-- existing link? -> use linked local user
        |
        +-- no link? -> apply account creation/linking policy
```

Do not automatically link accounts merely because two providers return the same email unless you have a carefully verified policy. Email verification semantics differ by provider.

Safer options:
- Ask the user to sign in to the existing account, then explicitly connect the new provider.
- Require a verified email and additional confirmation before linking.
- Notify the user when a new identity provider is connected.
- Allow users to remove a linked provider only if another usable sign-in method remains.

Use a database uniqueness constraint and handle concurrent first-time logins so two callbacks cannot create duplicate links.

## 10. Should social login issue your own JWT?

Usually, yes, if your API already uses JWT bearer authentication.

The provider authenticates the user. Your backend maps that identity to a local user and issues your application's credentials.

```text
Google ID token -> validate -> local User
                               |
                               v
                      your access token
                      your refresh token
                               |
                               v
                         FastAPI API
```

Your API should validate your own token's signature, issuer, audience, expiry, and relevant claims. Keep access tokens short-lived and design refresh-token rotation/revocation carefully.

Do not accept a Google access token as if it were your API's own access token.

## 11. Social login with React frontend

Two common designs:

### Design A: Backend owns the redirect flow

1. React sends the browser to `https://api.example.com/auth/google/login`.
2. FastAPI redirects to Google.
3. Google redirects to FastAPI callback.
4. FastAPI establishes a secure session or issues a short-lived one-time handoff to the frontend.

Avoid putting long-lived tokens in URL query strings. URLs can leak through browser history, logs, analytics, and referrer headers.

### Design B: Frontend uses an identity SDK

1. React uses the provider's approved SDK.
2. Provider authenticates the user.
3. React obtains an ID token or credential.
4. React sends it to FastAPI over HTTPS.
5. FastAPI verifies it with the provider's supported verification mechanism.
6. FastAPI finds/creates the local user and issues application credentials.

Use the provider's documented SDK and token validation rules. Do not trust a decoded but unverified frontend token.

For browser apps, consider a secure HttpOnly cookie-based session or a carefully designed token storage strategy. Avoid storing long-lived bearer tokens in localStorage when a safer architecture is practical.

## 12. Security checklist

- Use HTTPS in production.
- Validate OAuth `state` to prevent login CSRF.
- Validate OIDC `nonce` when applicable.
- Use PKCE where supported/appropriate.
- Validate ID-token signature, issuer, audience, expiry, and nonce.
- Use the provider's stable subject (`sub` or provider-specific ID) as the identity key.
- Do not assume an email is verified unless the provider says so and the claim is validated.
- Never expose client secrets in frontend code.
- Do not log access tokens, ID tokens, authorization codes, or secrets.
- Use exact redirect URIs.
- Restrict scopes to the minimum needed.
- Protect callback endpoints and account-linking actions.
- Rate-limit login initiation and sensitive account operations.
- Keep provider tokens only if you need to call provider APIs later; encrypt them at rest and manage refresh/revocation.
- Provide a safe logout and account unlinking policy.

## 13. Social authentication is not authorization

Authentication answers: **Who is this user?**

Authorization answers: **What may this user do?**

A Google-authenticated user is not automatically an administrator. After social login, load your local user's role and permissions from your own database.

Example:

```python
if not current_user.is_active:
    raise HTTPException(status_code=403, detail="Account disabled")

if current_user.role != "admin":
    raise HTTPException(status_code=403, detail="Not allowed")
```

Never trust a provider profile field such as display name or email to determine application privileges.

## 14. Which approach should you choose?

| Situation | Suggested approach |
|---|---|
| Learn OAuth/OIDC and control the flow | Authlib |
| Want integrated user-management abstractions | Evaluate fastapi-users and its current OAuth support |
| Want managed identity with less protocol code | Auth0, Clerk, or WorkOS |
| Want self-hosted identity server | Keycloak |
| Need enterprise Microsoft identity | Microsoft Entra ID with OIDC |

For a learning project, start with Google OIDC using Authlib. Then add GitHub OAuth. Once you understand the callback and local-account mapping, add Facebook and Microsoft.

## 15. Recommended implementation order

1. Create the local `User` table.
2. Create the `OAuthAccount` table with a unique provider identity constraint.
3. Implement Google login with Authlib.
4. Validate callback state and OIDC identity claims.
5. Find/create the local user.
6. Issue your application's own session or JWT.
7. Add logout and refresh/revocation behavior.
8. Add GitHub.
9. Add Facebook.
10. Add Microsoft.
11. Add explicit account linking/unlinking.
12. Test invalid state, expired code, denied consent, missing email, unverified email, duplicate callback, and provider outage.

## 16. Official starting points

- OpenID Connect: https://openid.net/developers/how-connect-works/
- OAuth 2.0 Security Best Current Practice: https://www.rfc-editor.org/rfc/rfc9700
- Authlib documentation: https://docs.authlib.org/
- FastAPI security: https://fastapi.tiangolo.com/tutorial/security/
- Google Identity: https://developers.google.com/identity
- GitHub OAuth: https://docs.github.com/en/apps/oauth-apps
- Meta Login: https://developers.facebook.com/docs/facebook-login/
- Microsoft identity platform: https://learn.microsoft.com/en-us/entra/identity-platform/

## 17. Final mental model

A social provider does not become your user database. It verifies an external identity. Your FastAPI application maps that external identity to a local user, applies its own authorization rules, and issues its own API session or JWT.
