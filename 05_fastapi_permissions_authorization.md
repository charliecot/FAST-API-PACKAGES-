# Permissions and Authorisation in FastAPI — From Basic Checks to RBAC

## 1. Authentication is not permission

Authentication identifies the caller. Permission logic decides what the caller can do.

Example:
- A valid token proves the request belongs to user 12.
- It does not automatically mean user 12 can delete every band's record.

A secure API checks permissions on every protected operation, including reads where data is private.

## 2. Common permission models

### A. Public access
Anyone can call the endpoint.

```python
@app.get("/public-bands")
def public_bands():
    ...
```

### B. Authenticated access
Any active authenticated user may call it.

```python
@app.get("/profile")
def profile(user=Depends(get_current_user)):
    ...
```

### C. Role-based access control (RBAC)
Users have roles such as `user`, `editor`, or `admin`. Roles grant groups of permissions.

### D. Ownership-based access
A user can access a record only if they own it.

### E. Attribute-based access control (ABAC)
Decisions depend on attributes such as owner, department, record state, time, or resource classification.

Real systems often combine RBAC and ownership checks.

## 3. A simple role field

```python
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    email: str
    hashed_password: str
    role: str = "user"
    is_active: bool = True
```

For a small learning project, a string role is understandable. For a larger system, consider an enum or separate Role/Permission tables, database constraints, and migrations.

Never allow an ordinary user to set their own role through a public registration or profile-update schema.

## 4. Build a reusable dependency

```python
from fastapi import Depends, HTTPException, status

def require_admin(current_user: User = Depends(get_current_user)):
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Administrator permission required",
        )
    return current_user
```

Use it:

```python
@app.delete("/users/{user_id}")
def delete_user(
    user_id: int,
    admin: User = Depends(require_admin),
):
    ...
```

This avoids repeating the same role check in every endpoint.

## 5. A flexible role dependency factory

```python
def require_roles(*allowed_roles: str):
    def role_checker(user: User = Depends(get_current_user)):
        if user.role not in allowed_roles:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user

    return role_checker
```

Usage:

```python
@app.post("/bands/")
def create_band(
    payload: BandCreate,
    user: User = Depends(require_roles("admin", "editor")),
):
    ...
```

`require_roles("admin", "editor")` returns a dependency function. FastAPI calls it and injects the current user.

## 6. Ownership permissions

Suppose a resource belongs to a user:

```python
class Article(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    title: str
    body: str
    owner_id: int = Field(foreign_key="user.id")
```

A user should only edit their own article:

```python
def get_article_for_owner(
    article_id: int,
    user: User,
    session: Session,
) -> Article:
    article = session.get(Article, article_id)

    if article is None:
        raise HTTPException(status_code=404, detail="Article not found")

    if article.owner_id != user.id:
        raise HTTPException(status_code=403, detail="You do not own this article")

    return article
```

Use it inside the endpoint:

```python
@app.patch("/articles/{article_id}")
def update_article(
    article_id: int,
    payload: ArticleUpdate,
    user: User = Depends(get_current_user),
    session: Session = Depends(get_session),
):
    article = get_article_for_owner(article_id, user, session)
    ...
```

An alternative is to include ownership in the database query:

```python
statement = select(Article).where(
    Article.id == article_id,
    Article.owner_id == user.id,
)
article = session.exec(statement).first()
```

This can reduce accidental exposure. Decide whether a non-owned record should return 403 or 404; 404 can avoid revealing that a protected record exists.

## 7. Never trust user_id from the request body

Unsafe pattern:

```python
@app.post("/articles/")
def create_article(payload: ArticleCreate):
    article = Article(**payload.model_dump())
```

If the payload includes `owner_id`, a caller may create an article owned by another user.

Safer:

```python
@app.post("/articles/")
def create_article(
    payload: ArticleCreate,
    user: User = Depends(get_current_user),
    session: Session = Depends(get_session),
):
    article = Article(
        **payload.model_dump(),
        owner_id=user.id,
    )
    session.add(article)
    session.commit()
    session.refresh(article)
    return article
```

Do not include `owner_id`, `role`, `is_admin`, or other privileged fields in public create/update schemas unless there is a carefully authorised administrative use case.

## 8. Permissions with scopes

OAuth2 scopes represent fine-grained permissions such as `bands:read` or `bands:write`. FastAPI supports OAuth2 scope declarations and `Security` dependencies.

Conceptually:

```python
from fastapi import Security
from fastapi.security import SecurityScopes

def get_current_user_with_scopes(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
):
    payload = decode_access_token(token)
    token_scopes = payload.get("scope", "").split()

    for required_scope in security_scopes.scopes:
        if required_scope not in token_scopes:
            raise HTTPException(status_code=403, detail="Not enough permissions")

    return load_user_from_subject(payload["sub"])
```

Endpoint:

```python
@app.get("/bands/")
def list_bands(
    user=Security(get_current_user_with_scopes, scopes=["bands:read"]),
):
    ...
```

This is a conceptual pattern; a production implementation must validate how scopes are granted, encoded, and revoked. Do not let clients choose their own scopes at login.

## 9. Router-level protection

You can protect an entire router:

```python
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_admin)],
)

@router.get("/reports")
def reports():
    return {"message": "Admin report"}
```

The dependency runs for routes in that router. If an endpoint also needs the user object, declare a dependency that returns it.

## 10. 401 vs 403

- **401 Unauthorized**: credentials are missing, invalid, or no longer acceptable.
- **403 Forbidden**: caller is authenticated but lacks permission.

Some frameworks and authentication schemes have nuances, but this is the useful API design distinction.

## 11. Object-level permission checks

A role check alone is not enough. For example, all users may have the `user` role, but each user should see only their own orders.

For each endpoint ask:
1. Who is calling?
2. Is the identity active?
3. What action is requested?
4. Which object is affected?
5. Does this user have permission for this exact object?
6. Can the requested fields be changed by this user?

This is especially important for IDs in URL paths. Guessing another user's ID must not grant access.

## 12. Test permission boundaries

Test at least:

- anonymous caller;
- authenticated ordinary user;
- owner of the object;
- authenticated non-owner;
- editor;
- administrator;
- inactive account;
- invalid or expired token.

Example:

```python
def test_non_admin_cannot_delete_user(client, normal_user_token):
    response = client.delete(
        "/users/10",
        headers={"Authorization": f"Bearer {normal_user_token}"},
    )
    assert response.status_code == 403
```

Also test that an owner cannot update protected fields such as `owner_id` or `role`.

## 13. Security mistakes to avoid

- Trusting role or owner IDs from request bodies.
- Checking permissions only in the frontend.
- Protecting writes but forgetting private reads.
- Using one broad `is_admin` check for every fine-grained operation.
- Returning private data before checking ownership.
- Forgetting permissions on nested resources.
- Assuming a valid JWT means the user is still active.
- Failing to revoke/refresh role information when roles change. If roles are embedded in JWT claims, old claims can remain valid until token expiry unless you re-check or revoke sessions.

## 14. Django REST Framework comparison

| Django REST Framework | FastAPI approach |
|---|---|
| `IsAuthenticated` | `Depends(get_current_user)` |
| `IsAdminUser` | `Depends(require_admin)` |
| Custom permission class | Dependency or explicit policy function |
| Object permission | Query/ownership check |
| Permission classes | Dependencies and policy functions |
| `request.user` | Injected current-user dependency |

FastAPI does not impose one permission-class system. Dependencies and ordinary Python functions let you build the policy you need.

## 15. Further reading

- FastAPI dependencies: https://fastapi.tiangolo.com/tutorial/dependencies/
- FastAPI OAuth2 scopes: https://fastapi.tiangolo.com/advanced/security/oauth2-scopes/
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- OWASP API Security Top 10: https://owasp.org/www-project-api-security/
