# Database Security Skills

## SQL Injection Prevention
```python
# NEVER — string concatenation
query = f"SELECT * FROM users WHERE email = '{user_input}'"  # DANGEROUS

# ALWAYS — parameterized queries
# psycopg2
cursor.execute("SELECT * FROM users WHERE email = %s", (user_input,))

# SQLAlchemy ORM
user = session.query(User).filter(User.email == user_input).first()

# SQLAlchemy Core with text()
from sqlalchemy import text
result = conn.execute(text("SELECT * FROM users WHERE email = :email"),
                      {"email": user_input})
```

## Principle of Least Privilege
```sql
-- Create read-only user for reporting
CREATE USER report_user WITH PASSWORD 'strong_pass';
GRANT CONNECT ON DATABASE mydb TO report_user;
GRANT USAGE ON SCHEMA public TO report_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_user;

-- App user — only what it needs
CREATE USER app_user WITH PASSWORD 'strong_pass';
GRANT SELECT, INSERT, UPDATE ON users, orders, products TO app_user;
REVOKE DELETE ON users FROM app_user;  -- app never deletes users directly
```

## Encryption at Rest
```python
from cryptography.fernet import Fernet
import os, base64

# Generate key (store in secret manager, NOT in code)
KEY = os.environ["DB_ENCRYPTION_KEY"].encode()
cipher = Fernet(KEY)

def encrypt(value: str) -> str:
    return cipher.encrypt(value.encode()).decode()

def decrypt(value: str) -> str:
    return cipher.decrypt(value.encode()).decode()

# In model — encrypt PII before storing
class User(Base):
    __tablename__ = "users"
    id    = Column(Integer, primary_key=True)
    _ssn  = Column("ssn", String)  # stored encrypted

    @property
    def ssn(self): return decrypt(self._ssn)
    @ssn.setter
    def ssn(self, v): self._ssn = encrypt(v)
```

## Password Hashing
```python
import bcrypt

def hash_password(plain: str) -> str:
    return bcrypt.hashpw(plain.encode(), bcrypt.gensalt(rounds=12)).decode()

def verify_password(plain: str, hashed: str) -> bool:
    return bcrypt.checkpw(plain.encode(), hashed.encode())

# NEVER: MD5, SHA1, SHA256 for passwords
# ALWAYS: bcrypt (rounds>=12), argon2id, scrypt
```

## Audit Logging
```sql
-- Audit table
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    table_name  VARCHAR(50),
    record_id   BIGINT,
    action      VARCHAR(10),  -- INSERT/UPDATE/DELETE
    old_data    JSONB,
    new_data    JSONB,
    changed_by  VARCHAR(100),
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger for automatic logging
CREATE OR REPLACE FUNCTION audit_trigger() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log(table_name, record_id, action, old_data, new_data, changed_by)
    VALUES(TG_TABLE_NAME,
           COALESCE(NEW.id, OLD.id),
           TG_OP,
           CASE WHEN TG_OP='INSERT' THEN NULL ELSE row_to_json(OLD) END,
           CASE WHEN TG_OP='DELETE' THEN NULL ELSE row_to_json(NEW) END,
           current_user);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

## Connection Security
```python
# Always use SSL for production DB connections
DATABASE_URL = "postgresql://user:pass@host:5432/db?sslmode=require"

# Connection pool with limits
from sqlalchemy import create_engine
engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600,  # recycle connections hourly
    connect_args={"sslmode": "require", "connect_timeout": 10},
)
```

## Backup & Recovery
```bash
# PostgreSQL backup
pg_dump -h localhost -U admin -d mydb -F c -f backup_$(date +%Y%m%d).dump

# Restore
pg_restore -h localhost -U admin -d mydb backup_20250401.dump

# Automated backup script
#!/bin/bash
BACKUP_DIR="/backups/postgres"
DB_NAME="mydb"
DATE=$(date +%Y%m%d_%H%M%S)
pg_dump -Fc $DB_NAME > "$BACKUP_DIR/${DB_NAME}_${DATE}.dump"
# Keep last 7 days
find $BACKUP_DIR -name "*.dump" -mtime +7 -delete
```

## Security Checklist
- [ ] All queries parameterized (no string concat with user input)
- [ ] DB user has minimum required permissions
- [ ] Passwords hashed with bcrypt/argon2 (never plaintext/MD5)
- [ ] PII fields encrypted at rest
- [ ] DB connection uses SSL/TLS
- [ ] Sensitive fields excluded from logs
- [ ] Backups encrypted and tested
- [ ] Audit log for sensitive table changes
- [ ] DB port not exposed to public internet
- [ ] Connection string in env var, not in code
