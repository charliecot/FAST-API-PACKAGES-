# CORS in FastAPI — Complete Practical Guide

## 1. The browser problem CORS solves

CORS means **Cross-Origin Resource Sharing**. It is a browser security mechanism that controls whether JavaScript running on one origin may read responses from another origin.

An origin consists of three parts:

```text
scheme + hostname + port
```

Examples:

```text
http://localhost:5173
http://localhost:8000
https://nzeggecharles.org
```

These are different origins because the port, scheme, or host differs.

A common development setup:

- React/Vite frontend: `http://localhost:5173`
- FastAPI backend: `http://localhost:8000`

The browser treats them as different origins even though both use `localhost`.

## 2. What CORS does — and does not do

CORS tells a browser whether frontend JavaScript is allowed to read a cross-origin response.

CORS is **not**:
- Authentication.
- Authorisation or permissions.
- A firewall.
- A way to stop curl, Postman, Python requests, or server-to-server calls.
- A substitute for CSRF protection.

A request may reach the server even when the browser refuses to expose its response to JavaScript.

## 3. Configure CORS in FastAPI

FastAPI uses Starlette's `CORSMiddleware`.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

origins = [
    "http://localhost:5173",
    "http://127.0.0.1:5173",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### Explanation of each option

- `allow_origins`: exact origins allowed to access the API from browsers.
- `allow_credentials`: whether browser requests may include credentials such as cookies or HTTP authentication information.
- `allow_methods`: HTTP methods allowed cross-origin.
- `allow_headers`: request headers the browser may send.
- `expose_headers`: response headers browser JavaScript is allowed to read when they are not CORS-safelisted.
- `max_age`: how long browsers may cache the result of a preflight permission response.

Example with more options:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],
    max_age=600,
)
```

## 4. What is a preflight request?

For some cross-origin requests, the browser first sends an automatic **preflight** request using HTTP OPTIONS.

For example, a frontend sends:

```javascript
fetch("http://localhost:8000/bands/1", {
  method: "PATCH",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer example-token"
  },
  body: JSON.stringify({ genre: "Jazz" })
});
```

The browser may first send something conceptually like:

```http
OPTIONS /bands/1 HTTP/1.1
Origin: http://localhost:5173
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: authorization,content-type
```

The server's middleware checks the origin, method, and requested headers. It responds with CORS permission headers if allowed.

The browser then sends the actual PATCH request.

A simple GET request with safelisted headers may not need preflight. A JSON content type, custom header, or method such as PATCH commonly causes preflight.

## 5. Important CORS response headers

| Header | Purpose |
|---|---|
| `Access-Control-Allow-Origin` | Which origin may read the response |
| `Access-Control-Allow-Methods` | Methods allowed by preflight |
| `Access-Control-Allow-Headers` | Request headers allowed by preflight |
| `Access-Control-Allow-Credentials` | Whether credentials are permitted |
| `Access-Control-Expose-Headers` | Additional response headers visible to JavaScript |
| `Access-Control-Max-Age` | Preflight cache duration |

The browser controls the `Origin` request header. Frontend JavaScript cannot freely choose an arbitrary Origin header.

## 6. Wildcard origin and credentials

This is a common mistake:

```python
allow_origins=["*"]
allow_credentials=True
```

Credentialed CORS requests require a specific allowed origin rather than a wildcard response origin. Browsers reject an invalid wildcard/credentials combination.

For an API that uses cookies, specify trusted origins explicitly.

For a public API that does not use credentials, a wildcard may be suitable:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["GET"],
    allow_headers=["*"],
)
```

Do not use permissive settings automatically in production.

## 7. Cookies, JWT, and CORS

JWT can be transported in different ways:

### Authorization header

```http
Authorization: Bearer <token>
```

Frontend:

```javascript
fetch("http://localhost:8000/private", {
  headers: {
    Authorization: `Bearer ${accessToken}`
  }
});
```

This usually requires allowing the `Authorization` header in CORS.

### Cookie-based authentication

If the browser must send cross-origin cookies:

```javascript
fetch("https://api.example.com/private", {
  credentials: "include"
});
```

The server must permit credentials and the exact frontend origin:

```python
allow_origins=["https://frontend.example.com"]
allow_credentials=True
```

Cookie attributes also matter:
- `HttpOnly`: prevents JavaScript from reading the cookie.
- `Secure`: sends it only over secure HTTPS connections.
- `SameSite`: controls cross-site cookie behaviour (`Strict`, `Lax`, or `None`).
- `SameSite=None` requires `Secure` in modern browsers.

Cross-site cookie authentication also needs a CSRF strategy. CORS alone does not prevent CSRF.

## 8. CORS and CSRF are different

CORS controls whether browser JavaScript can read a cross-origin response. CSRF concerns unwanted state-changing requests made using a victim's automatically attached credentials, often cookies.

If using cookie authentication:
- use appropriate `SameSite` settings;
- use CSRF tokens or another robust CSRF defence where required;
- validate origins/referers as an additional defence where appropriate;
- use HTTPS.

A bearer token manually attached in an Authorization header has a different CSRF profile, but token storage and XSS risks still matter.

## 9. Development vs production

Development:

```python
origins = [
    "http://localhost:5173",
    "http://127.0.0.1:5173",
]
```

Production:

```python
origins = [
    "https://nzeggecharles.org",
    "https://www.nzeggecharles.org",
]
```

Do not assume `localhost` and `127.0.0.1` are the same origin. They are different hostnames.

For multiple environments, load allowed origins from environment configuration rather than hardcoding secrets or environment-specific values throughout your code.

## 10. Debugging CORS errors

When a browser reports a CORS error:

1. Read the browser console message.
2. Open Developer Tools → Network.
3. Inspect the OPTIONS request if one exists.
4. Check the request's `Origin`.
5. Check the server response's `Access-Control-Allow-Origin`.
6. Check whether the requested method and headers are permitted.
7. Check credentials and cookie settings.
8. Check whether a proxy, CDN, or reverse proxy is removing headers.
9. Check the actual backend logs; a server error can sometimes be misreported as a CORS problem.

Common causes:
- Wrong port in `allow_origins`.
- Missing `http` vs `https`.
- Trailing slash mistakenly included in an origin. An origin is a scheme/host/port tuple, not a full endpoint path.
- `Authorization` not permitted.
- PATCH or DELETE not permitted.
- Credentials enabled with wildcard origin.
- OPTIONS blocked by a proxy or custom middleware.
- Frontend calling a different hostname than expected.

## 11. Test outside the browser

CORS is primarily enforced by browsers. A successful curl request does not prove browser CORS is configured correctly.

Inspect headers:

```bash
curl -i \
  -H "Origin: http://localhost:5173" \
  http://localhost:8000/bands/
```

Simulate preflight:

```bash
curl -i -X OPTIONS \
  http://localhost:8000/bands/1 \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: PATCH" \
  -H "Access-Control-Request-Headers: authorization,content-type"
```

Check that the response contains the expected CORS headers.

## 12. Middleware order and application layout

Add middleware to the FastAPI application object that actually serves the routes. If you create multiple FastAPI instances, configuring CORS on the wrong instance will not help.

A typical layout:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routers import bands, albums

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(bands.router)
app.include_router(albums.router)
```

## 13. Security checklist

- [ ] Explicit trusted origins in production.
- [ ] No wildcard origin with credentialed requests.
- [ ] Only required methods and headers allowed.
- [ ] HTTPS in production.
- [ ] CSRF protection when cookie authentication requires it.
- [ ] CORS is not treated as authentication.
- [ ] Test actual browser requests, including preflight.
- [ ] Configure the correct FastAPI app instance.
- [ ] Review proxy/CDN behaviour.

## 14. Official documentation

- FastAPI CORS: https://fastapi.tiangolo.com/tutorial/cors/
- Starlette middleware: https://www.starlette.io/middleware/#corsmiddleware
- MDN CORS: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
