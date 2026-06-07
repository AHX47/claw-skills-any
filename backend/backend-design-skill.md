# Backend Design Reading Skill

## How to Read Any Backend Codebase

### Step 1 — Entry Points
```
FastAPI/Flask:   app = FastAPI()  →  uvicorn.run()
Django:          wsgi.py / asgi.py  →  manage.py runserver
Express:         app.listen(port)
Spring Boot:     @SpringBootApplication  →  main()
Laravel:         public/index.php  →  bootstrap/app.php
```

### Step 2 — Follow the Request
```
HTTP Request
    → Router/Controller   (what URL maps to what handler?)
    → Middleware/Guards   (auth, rate-limit, logging)
    → Service Layer       (business logic)
    → Repository/DAO      (data access)
    → Database/Cache
    → Response
```

### Step 3 — Identify Layers
| Layer | What It Does | Files to Look For |
|-------|-------------|------------------|
| Presentation | HTTP in/out | routes/, controllers/, views/ |
| Application | Use cases | services/, usecases/, handlers/ |
| Domain | Business rules | models/, entities/, domain/ |
| Infrastructure | DB, cache, queue | repositories/, adapters/, db/ |

---

## Common Patterns You'll Encounter

### Repository Pattern
```python
class UserRepository(ABC):
    @abstractmethod
    def find_by_id(self, id: int) -> User | None: ...
    @abstractmethod
    def save(self, user: User) -> User: ...

class PostgresUserRepository(UserRepository):
    def find_by_id(self, id: int) -> User | None:
        row = self.db.execute("SELECT * FROM users WHERE id=$1", [id]).fetchone()
        return User.from_row(row) if row else None
```

### Service Layer
```python
class UserService:
    def __init__(self, repo: UserRepository, mailer: Mailer, cache: Cache):
        self.repo   = repo
        self.mailer = mailer
        self.cache  = cache

    def register(self, dto: RegisterDto) -> User:
        if self.repo.find_by_email(dto.email):
            raise ConflictError("Email already registered")
        user = User.create(dto.name, dto.email, hash_password(dto.password))
        self.repo.save(user)
        self.mailer.send_welcome(user.email)
        self.cache.invalidate(f"users:{user.id}")
        return user
```

### Dependency Injection
```python
# FastAPI DI
from functools import lru_cache
from fastapi import Depends

@lru_cache
def get_settings() -> Settings:
    return Settings()

def get_db(settings: Settings = Depends(get_settings)) -> Database:
    return Database(settings.db_url)

def get_user_service(db: Database = Depends(get_db)) -> UserService:
    return UserService(PostgresUserRepository(db))

@router.get("/users/{id}")
async def get_user(id: int, svc: UserService = Depends(get_user_service)):
    return svc.find_by_id(id)
```

---

## Database Patterns

### Migrations
```sql
-- Always: timestamp, description, up/down
-- 20250401_001_create_users.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,
    role       VARCHAR(20)  NOT NULL DEFAULT 'user',
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
```

### Query Optimization
```sql
-- Use EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT u.*, p.* FROM users u
JOIN profiles p ON p.user_id = u.id
WHERE u.email = 'test@example.com';

-- Add index for join + filter columns
CREATE INDEX idx_profiles_user_id ON profiles(user_id);

-- Avoid N+1: use JOIN or batch fetch
-- BAD:  for each user: SELECT * FROM orders WHERE user_id=?
-- GOOD: SELECT * FROM orders WHERE user_id IN (1,2,3,4,5)
```

---

## API Design Principles

### REST Conventions
```
GET    /users          → list users
GET    /users/{id}     → get one user
POST   /users          → create user
PATCH  /users/{id}     → partial update
PUT    /users/{id}     → full replace
DELETE /users/{id}     → delete

GET    /users/{id}/orders   → user's orders (nested resource)
POST   /users/{id}/activate → action (verb when needed)
```

### Response Structure
```json
// Success
{ "data": {...}, "meta": { "total": 100, "page": 1 } }

// Error
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }

// Paginated list
{ "data": [...], "meta": { "total": 250, "page": 2, "per_page": 20, "pages": 13 } }
```

### HTTP Status Codes
```
200 OK             → GET success
201 Created        → POST success
204 No Content     → DELETE success
400 Bad Request    → validation error
401 Unauthorized   → not authenticated
403 Forbidden      → authenticated but no permission
404 Not Found      → resource missing
409 Conflict       → duplicate, version conflict
422 Unprocessable  → semantic validation error
429 Too Many Req   → rate limited
500 Internal Error → unexpected server error
```

---

## Auth Patterns

### JWT Flow
```python
import jwt, datetime

SECRET = os.environ["JWT_SECRET"]
ALGO   = "HS256"

def create_token(user_id: int, role: str) -> str:
    payload = {
        "sub":  str(user_id),
        "role": role,
        "iat":  datetime.datetime.utcnow(),
        "exp":  datetime.datetime.utcnow() + datetime.timedelta(hours=24),
    }
    return jwt.encode(payload, SECRET, algorithm=ALGO)

def verify_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET, algorithms=[ALGO])
    except jwt.ExpiredSignatureError:
        raise AuthError("Token expired")
    except jwt.InvalidTokenError:
        raise AuthError("Invalid token")
```

### Rate Limiting
```python
from collections import defaultdict
import time

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max = max_requests
        self.window = window_seconds
        self._buckets: dict[str, list[float]] = defaultdict(list)

    def allow(self, key: str) -> bool:
        now = time.time()
        bucket = self._buckets[key]
        # Remove old timestamps
        self._buckets[key] = [t for t in bucket if now - t < self.window]
        if len(self._buckets[key]) >= self.max:
            return False
        self._buckets[key].append(now)
        return True
```

---

## Reading Checklist
- [ ] What framework/version is used?
- [ ] Where is the entry point?
- [ ] How is configuration loaded? (env, files, DI)
- [ ] What database(s) are used? ORM or raw SQL?
- [ ] How is authentication handled? (JWT, session, OAuth)
- [ ] Where is business logic? (service layer vs controllers)
- [ ] How are errors handled and formatted?
- [ ] Is there a job queue? (Celery, BullMQ, Sidekiq)
- [ ] What external services are integrated? (email, storage, payments)
- [ ] Are there background tasks or scheduled jobs?
