# 07 — Authentication and Security

> **Goal**: Implement OAuth2 password flow with JWT tokens, password hashing, and protected routes in FastAPI.

---

## 🧠 How Authentication Works in APIs

```
1. Client sends username + password to /login
2. Server verifies credentials against the database
3. Server creates a JWT token and sends it back
4. Client stores the token (localStorage, cookie, etc.)
5. Client sends the token in every subsequent request:
     Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
6. Server verifies the token and identifies the user

┌────────┐                              ┌────────┐
│ Client │ ── POST /login ──────────→   │ Server │
│        │    {username, password}       │        │
│        │                              │        │
│        │ ←── {access_token: "eyJ..."} │        │
│        │                              │        │
│        │ ── GET /protected ────────→  │        │
│        │    Authorization: Bearer ... │        │
│        │                              │        │
│        │ ←── {data: "secret stuff"}   │        │
└────────┘                              └────────┘
```

---

## 📦 Install Dependencies

```bash
pip install "python-jose[cryptography]"  # JWT token creation/verification
pip install "passlib[bcrypt]"            # Password hashing
pip install python-multipart             # For OAuth2 form data
```

| Package            | Purpose                                                  |
| ------------------ | -------------------------------------------------------- |
| `python-jose`      | Create and verify JWT (JSON Web Tokens)                  |
| `passlib`          | Hash passwords with bcrypt (industry standard)           |
| `python-multipart` | Parse form data (OAuth2 login sends form data, not JSON) |

---

## 🔐 Password Hashing

**Never store plain text passwords.** Always hash them.

```python
# app/security.py

from passlib.context import CryptContext

# Create a password hashing context
# bcrypt is the recommended algorithm for password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
# schemes=["bcrypt"]: Use bcrypt for hashing
# deprecated="auto": Automatically handle old hash formats

def hash_password(password: str) -> str:
    """
    Converts "mypassword123" into something like:
    "$2b$12$LJ3m4ys3Lg2kF45K.sPRKOzqNIBnSMzJLFm..."

    This is a ONE-WAY operation. You cannot get "mypassword123"
    back from the hash. You can only VERIFY if a password matches.

    bcrypt is intentionally SLOW (~100ms per hash). This makes
    brute-force attacks impractical.
    """
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Checks if a plain text password matches a hashed password.
    Returns True if they match, False otherwise.

    Internally, bcrypt:
    1. Extracts the salt from the hashed_password
    2. Hashes plain_password with the same salt
    3. Compares the two hashes
    """
    return pwd_context.verify(plain_password, hashed_password)

# Example usage:
# hashed = hash_password("secret123")
# verify_password("secret123", hashed)   → True
# verify_password("wrongpass", hashed)    → False
```

---

## 🎟️ JWT Tokens

```python
# app/security.py (continued)

from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone

# Secret key for signing tokens
# In production, use a long random string stored in environment variables!
# Generate one with: openssl rand -hex 32
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"

# Algorithm for JWT signing
ALGORITHM = "HS256"
# HS256 = HMAC with SHA-256
# This means the token is signed (not encrypted) with our secret key

ACCESS_TOKEN_EXPIRE_MINUTES = 30


def create_access_token(data: dict, expires_delta: timedelta = None) -> str:
    """
    Creates a JWT token containing the given data.

    A JWT has three parts (separated by dots):
    1. HEADER:  {"alg": "HS256", "typ": "JWT"}  (base64 encoded)
    2. PAYLOAD: {"sub": "john", "exp": 1709856000}  (base64 encoded)
    3. SIGNATURE: HMAC-SHA256(header + "." + payload, SECRET_KEY)

    The signature ensures no one can tamper with the payload.
    If they change the payload, the signature won't match.

    Example token:
    eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwiZXhwIjoxNzA5ODU2MDAwfQ.abc123...
    """
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire})
    # "exp" is a standard JWT claim — the token is invalid after this time

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt


def verify_access_token(token: str) -> dict:
    """
    Decodes and verifies a JWT token.

    1. Splits the token into header, payload, signature
    2. Recalculates the signature using SECRET_KEY
    3. Compares with the provided signature
    4. Checks if the token has expired (exp claim)
    5. Returns the payload if valid, raises JWTError if not
    """
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None
```

---

## 🏗️ Putting It All Together

### Pydantic Schemas for Auth

```python
# app/schemas.py (add these)

from pydantic import BaseModel
from typing import Optional

class Token(BaseModel):
    """Response schema for the login endpoint."""
    access_token: str
    token_type: str  # Always "bearer"

class TokenData(BaseModel):
    """Data extracted from a JWT token."""
    username: Optional[str] = None

class UserLogin(BaseModel):
    """Login request body (used internally)."""
    username: str
    password: str
```

### OAuth2 Setup and Dependencies

```python
# app/auth.py

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from app.security import verify_password, verify_access_token
from app.database import SessionLocal
from app import models

# ─── OAuth2 Scheme ───
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")
# tokenUrl="login" tells the Swagger UI:
#   "To get a token, send a POST request to /login"
# The Swagger docs will show a 🔒 button for authentication.
#
# What OAuth2PasswordBearer does:
# 1. Looks for the "Authorization" header in the request
# 2. Checks that it starts with "Bearer "
# 3. Extracts the token (everything after "Bearer ")
# 4. Returns the token string
# 5. If no header → returns 401 Unauthorized


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


async def get_current_user(
    token: str = Depends(oauth2_scheme),  # Extracts token from header
    db: Session = Depends(get_db)
):
    """
    This dependency:
    1. Gets the JWT token from the Authorization header
    2. Verifies and decodes the token
    3. Looks up the user in the database
    4. Returns the user object

    If anything fails, it raises 401 Unauthorized.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
        # WWW-Authenticate header tells the client it needs a Bearer token
    )

    # Verify the token
    payload = verify_access_token(token)
    if payload is None:
        raise credentials_exception

    username: str = payload.get("sub")
    # "sub" (subject) is a standard JWT claim — we use it for the username
    if username is None:
        raise credentials_exception

    # Look up the user in the database
    user = db.query(models.User).filter(models.User.username == username).first()
    if user is None:
        raise credentials_exception

    return user


async def get_current_active_user(
    current_user: models.User = Depends(get_current_user)
):
    """
    Extra check: is the user account active?
    This is a nested dependency — it depends on get_current_user.
    """
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user
```

### Login and Protected Endpoints

```python
# app/main.py (add these routes)

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from datetime import timedelta
from app.auth import get_current_active_user, get_db
from app.security import verify_password, create_access_token, hash_password
from app import models, schemas

app = FastAPI()

# ────── Login Endpoint ──────
@app.post("/login", response_model=schemas.Token)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """
    OAuth2PasswordRequestForm automatically parses:
    - form_data.username (from form field)
    - form_data.password (from form field)

    Note: OAuth2 uses FORM DATA, not JSON!
    Content-Type: application/x-www-form-urlencoded

    The Swagger UI (/docs) provides a login form for this.
    """
    # 1. Find the user
    user = db.query(models.User).filter(
        models.User.username == form_data.username
    ).first()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # 2. Verify the password
    if not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
        )

    # 3. Create a JWT token
    access_token = create_access_token(
        data={"sub": user.username},  # "sub" = subject = who this token is for
        expires_delta=timedelta(minutes=30)
    )

    return {"access_token": access_token, "token_type": "bearer"}


# ────── Register Endpoint ──────
@app.post("/register", response_model=schemas.UserResponse, status_code=201)
def register(user: schemas.UserCreate, db: Session = Depends(get_db)):
    # Check if user exists
    existing = db.query(models.User).filter(
        models.User.username == user.username
    ).first()
    if existing:
        raise HTTPException(status_code=400, detail="Username taken")

    # Create user with HASHED password
    db_user = models.User(
        username=user.username,
        email=user.email,
        hashed_password=hash_password(user.password),  # Hash it!
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user


# ────── Protected Endpoints ──────
@app.get("/me", response_model=schemas.UserResponse)
def read_current_user(
    current_user: models.User = Depends(get_current_active_user)
):
    """
    This endpoint requires authentication.

    The dependency chain:
    1. oauth2_scheme → extracts token from "Authorization: Bearer ..."
    2. get_current_user → verifies token, looks up user in DB
    3. get_current_active_user → checks user.is_active
    4. This endpoint receives the authenticated user object

    If the token is missing, invalid, or expired → 401 Unauthorized
    """
    return current_user


@app.get("/protected-data")
def get_protected_data(
    current_user: models.User = Depends(get_current_active_user)
):
    return {
        "message": f"Hello {current_user.username}!",
        "secret": "This data is only visible to authenticated users"
    }
```

### Complete Auth Flow

```
1. Register:
   POST /register
   Body: {"username": "john", "email": "john@example.com", "password": "secret"}
   → Creates user with hashed password

2. Login:
   POST /login
   Form: username=john&password=secret
   → Returns: {"access_token": "eyJ...", "token_type": "bearer"}

3. Access protected route:
   GET /me
   Headers: Authorization: Bearer eyJ...
   → Returns: {"username": "john", "email": "john@example.com", ...}

4. Without token:
   GET /me
   → 401 Unauthorized: {"detail": "Not authenticated"}

5. With expired/invalid token:
   GET /me
   Headers: Authorization: Bearer invalid...
   → 401 Unauthorized: {"detail": "Could not validate credentials"}
```

---

## 🔬 Internals: How OAuth2PasswordBearer Works

```python
# OAuth2PasswordBearer is a class that:
#
# 1. Inherits from SecurityBase (Starlette)
# 2. Implements __call__ method, making it a callable dependency
#
# Simplified internal implementation:
class OAuth2PasswordBearer:
    def __init__(self, tokenUrl: str):
        self.tokenUrl = tokenUrl
        # tokenUrl is stored for OpenAPI schema generation
        # The Swagger UI uses it to know where to send login requests

    async def __call__(self, request: Request) -> str:
        # 1. Get the Authorization header
        authorization = request.headers.get("Authorization")

        if not authorization:
            raise HTTPException(status_code=401)

        # 2. Check it starts with "Bearer "
        scheme, _, token = authorization.partition(" ")
        if scheme.lower() != "bearer":
            raise HTTPException(status_code=401)

        # 3. Return just the token part
        return token

# When used with Depends(), FastAPI calls this for every request
# to a protected endpoint, extracting the token automatically.
```

---

## ➡️ Next Steps

Head to **[08 — Testing and Deployment](./08-testing-and-deployment.md)** to learn how to test your API with pytest and deploy it with Docker and Uvicorn.
