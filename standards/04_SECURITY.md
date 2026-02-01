# Security Guidelines

Security is a top priority. These guidelines are mandatory across all projects.

## Core Principles

1. **Never hardcode secrets** - Use environment variables or secure vaults
2. **Input validation always** - Validate all user input
3. **Least privilege** - Use minimum necessary permissions
4. **Defense in depth** - Multiple layers of protection
5. **Log security events** - Track suspicious activity

## Secrets & Credentials Management

### ✅ DO THIS

```bash
# Use environment variables
DATABASE_URL="postgresql://user:pass@localhost:5432/dbname"
API_SECRET_KEY="your-secret-key-here"

# Use .env files locally (NEVER commit)
# .env (local only)
# .env.example (safe template)
```

### ❌ NEVER DO THIS

- Commit secrets to version control
- Hardcode credentials in source code
- Share credentials in Slack, email, or chat
- Use production secrets in development
- Log sensitive data

### Ansible Vault Pattern

```yaml
---
- name: Deploy with vault
  hosts: servers
  vars:
    db_password: "{{ lookup('env', 'DB_PASSWORD') | default('') }}"
  
  pre_tasks:
    - name: Validate credentials
      assert:
        that:
          - db_password != ''
        fail_msg: "DB_PASSWORD not set in environment"
```

## Input Validation

### Database Queries

```python
# ✅ SAFE - Parameterized queries
result = db.execute(
    "SELECT * FROM users WHERE email = $1",
    email  # Parameter, never concatenated
)

# ❌ DANGEROUS - SQL injection risk
result = db.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

### API Endpoints

```python
# ✅ SAFE - Validate with Pydantic
from pydantic import BaseModel, validator, EmailStr

class UserInput(BaseModel):
    email: EmailStr
    name: str
    
    @validator('name')
    def validate_name(cls, v):
        if len(v) < 2 or len(v) > 100:
            raise ValueError('Name must be 2-100 characters')
        return v.strip()

@app.post("/users")
async def create_user(user: UserInput):
    # Data already validated
    return db.create_user(user.email, user.name)
```

### Frontend Validation

```javascript
// ✅ SAFE - Validate data from API
const response = await fetch('/api/user');
const data = await response.json();

if (typeof data.email !== 'string' || !data.email.includes('@')) {
    throw new Error('Invalid response format');
}
```

## Authentication & Authorization

### JWT Tokens

```python
JWT_SECRET_KEY = os.getenv("JWT_SECRET_KEY")  # From environment only
JWT_ALGORITHM = "HS256"
JWT_EXPIRATION_HOURS = 24

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, JWT_SECRET_KEY, algorithms=[JWT_ALGORITHM])
        user_id = payload.get("sub")
        if not user_id:
            raise HTTPException(status_code=401, detail="Invalid token")
        return await get_user_by_id(user_id)
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### Requirements
- Strong, randomly generated secret keys
- Appropriate token expiration (not "never")
- Token refresh mechanisms
- Validation on every protected endpoint
- Secure client-side storage (httpOnly cookies preferred)

## HTTPS & Transport Security

### Production Requirements

```python
# Force HTTPS in production
if ENVIRONMENT == "production":
    app.add_middleware(HTTPSRedirectMiddleware)

# Secure CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Specific origins only
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

## Error Handling

### ✅ SAFE - Hide internals from users

```python
@app.exception_handler(Exception)
async def generic_handler(request, exc):
    # Log detailed error for developers
    logger.error(f"Error: {exc}", exc_info=True)
    
    # Return generic message to users
    return JSONResponse(
        status_code=500,
        content={"error": "An internal error occurred"}
    )
```

### ❌ DANGEROUS - Exposing internals

```python
# Don't do this!
return JSONResponse(
    status_code=500,
    content={"error": f"Database connection failed: {str(exc)}"}  # Stack trace exposed!
)
```

## Dependency Management

### Keep Dependencies Secure

```bash
# Audit for vulnerabilities
pip-audit
npm audit

# Update regularly
pip list --outdated
npm outdated

# Pin versions in production
# requirements.txt should have exact versions
# poetry.lock / package-lock.json for reproducible builds
```

## Ansible-Specific Security

### Privilege Escalation

```yaml
# ✅ GOOD - Only when necessary
- name: Install system package
  package:
    name: nginx
    state: present
  become: true  # Documented why

# ❌ BAD - Over-privileged
- name: Create home directory file
  file:
    path: /home/user/.bashrc
    state: file
  become: true  # Not necessary!
```

### Credential Validation

```yaml
- name: Validate credentials before deployment
  block:
    - assert:
        that:
          - ansible_user_id is defined
          - vault_password is defined
        fail_msg: "Required credentials not available"
    
    - name: Deploy with validated credentials
      command: deploy.sh
```

## Security Checklist

### Before Every Release

- [ ] All secrets in environment variables only
- [ ] Input validation on all endpoints
- [ ] SQL injection protection verified
- [ ] Authentication/authorization tested
- [ ] HTTPS enforced in production
- [ ] Error handling doesn't expose internals
- [ ] Dependencies scanned for vulnerabilities
- [ ] Security headers configured

### Regular Maintenance

- [ ] Rotate credentials quarterly
- [ ] Update dependencies monthly
- [ ] Audit access permissions
- [ ] Review logs for suspicious activity
- [ ] Test backup/recovery procedures

## Common Vulnerability Patterns to Avoid

| Vulnerability | Risk | Prevention |
|---------------|------|-----------|
| SQL Injection | Critical | Parameterized queries only |
| Hardcoded Secrets | Critical | Environment variables + Vault |
| Unvalidated Input | High | Validate + sanitize all input |
| Weak Passwords | High | Enforce strong requirements |
| Missing Auth | Critical | Authenticate every endpoint |
| HTTPS Missing | High | HTTPS-only in production |
| Error Messages | Medium | Generic messages to users |
| Dependency Vulnerabilities | Medium-High | Regular audits + updates |

## Incident Response

If a security issue is discovered:

1. **Assess** - Scope and impact evaluation
2. **Contain** - Prevent further exposure
3. **Fix** - Apply security patches
4. **Document** - Record incident details
5. **Learn** - Update processes to prevent recurrence

---

**When in doubt, ask for a security review. It's better to be cautious.**

