# Role-Based Access Control (RBAC)

This extension adds user authentication, role management, and access control to your application. It allows you to control who can access different parts of your application based on their assigned roles (like admin, user, moderator).

## What This Does

RBAC provides:
- User database with authentication
- Role management (ROOT, ADMIN, USER roles)
- JWT token-based authentication
- User profile management
- Protected API endpoints based on roles

## Prerequisites

- This extension requires a database (use with extension-rdbms)
- Works with Spring Boot or Django REST Framework backends

---

## Configuration Fields

### Scaffold Configuration

#### Enable JWT Authentication `enableJwtAuth`
**What it is**: Determines whether to use JWT (JSON Web Token) for authentication instead of sessions.

**Options**:
- `true` (default): Use JWT tokens for stateless authentication (recommended for modern APIs)
- `false`: Use traditional session-based authentication

**When to choose JWT (true)**:
- Building a REST API
- Need mobile app support
- Want stateless authentication
- Planning microservices architecture

**When to choose sessions (false)**:
- Building a traditional web application
- Don't need mobile app support
- Prefer simpler session management

**What's the difference**:
- **JWT**: Token sent with each request, no server-side session storage
- **Sessions**: Session ID cookie, server stores session data

---

### Runtime Configuration

#### JWT Secret Key `JWT_SECRET_KEY`
**What it is**: A secret password used to sign and verify JWT tokens. This ensures tokens can't be forged.

**How to generate it**:
Option 1 - Use online generator:
- Go to https://randomkeygen.com
- Copy a "CodeIgniter Encryption Key" (256-bit)

Option 2 - Use command line:
```bash
# On Mac/Linux:
openssl rand -base64 32

# Or use Node.js:
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

**Format**: A long random string (at least 256 bits / 32 bytes)

**Example**: `Kv8J3N9mQ2wX5yZ7aB1cD4eF6gH8iK0l`

**Important**:
- **Keep this absolutely secret!**
- Different value for each environment (production, staging, etc.)
- Never commit to version control
- If compromised, all tokens become invalid when you change it

---

#### JWT Expiration `JWT_EXPIRATION_MS`
**What it is**: How long (in milliseconds) before a JWT token expires and user must log in again.

**Common values**:
- `3600000` = 1 hour (most secure)
- `86400000` = 24 hours (1 day) - good balance
- `604800000` = 7 days (convenient but less secure)
- `2592000000` = 30 days (least secure, most convenient)

**Default**: Usually 24 hours (86400000 ms)

**How to choose**:
- **Banking/sensitive apps**: 15-60 minutes
- **Business apps**: 1-8 hours  
- **Social/consumer apps**: 24 hours to 7 days
- **Internal tools**: 7-30 days

**Note**: Shorter expiration = more secure but users must log in more often

---

#### JWT Issuer URI `JWT_ISSUER_URI`
**What it is**: The URL that identifies who issued the JWT token. Used when integrating with OAuth2 providers (like Auth0, Okta).

**When to use**:
- Using with extension-auth0: Set to your Auth0 domain (e.g., `https://myapp.auth0.com/`)
- Using internal authentication: Can leave empty or set to your API URL

**Format**: A URL starting with `https://`

**Example**: 
- Auth0: `https://mycompany.auth0.com/`
- Okta: `https://mycompany.okta.com/`
- Internal: `https://api.myapp.com`

**Note**: This is optional for internal authentication, required for OAuth2 integration.

---

#### JWT JWK Set URI `JWT_JWK_SET_URI`
**What it is**: The URL where public keys are published for verifying JWT tokens. Used with OAuth2 providers.

**When to use**:
- Using with extension-auth0: Auth0 provides this URL
- Using with external OAuth provider: They provide this URL
- Using internal authentication: Leave empty

**Format**: URL ending in `/.well-known/jwks.json`

**Example**: 
- Auth0: `https://mycompany.auth0.com/.well-known/jwks.json`
- Okta: `https://mycompany.okta.com/oauth2/default/v1/keys`

**Note**: Only needed when using external OAuth2 authentication providers.

---

#### Root Users `APP_ROOT_USERS`
**What it is**: Email addresses of users who should have ROOT permissions (highest level - full system access).

**Format**: Comma-separated list of email addresses

**Example**: `founder@myapp.com,cto@myapp.com`

**Root users can**:
- Access everything
- Manage all users and roles
- Access admin panels
- Make system-wide changes

**Important**:
- Usually just 1-2 people (founders, CTO)
- These users are automatically assigned ROOT role on first login
- Be very careful who you add here
- Different per environment (production ROOT users ≠ staging ROOT users)

---

#### Admin Users `APP_ADMIN_USERS`
**What it is**: Email addresses of users who should have ADMIN permissions (high level - can manage users and content).

**Format**: Comma-separated list of email addresses

**Example**: `admin@myapp.com,support@myapp.com,manager@myapp.com`

**Admin users can**:
- Manage regular users
- Access admin features
- View analytics and reports
- Moderate content

**Admin users cannot**:
- Change ROOT users
- Access system-level settings
- Deploy or change infrastructure

**Use case**: Team members who need elevated permissions but not full system access.

---

## Role Hierarchy

**ROOT** (Superuser):
- Full system access
- Can do everything
- Assign: Founders, CTO

**ADMIN** (Administrator):
- Manage users and content
- Access admin panel
- Assign: Managers, support team

**USER** (Regular User):
- Basic access
- Own profile only
- Assign: All regular users (default)

**Note**: ROOT includes ADMIN permissions, ADMIN includes USER permissions.

---

## How It Works

### User Creation
1. User signs up or logs in
2. System checks if their email is in `APP_ROOT_USERS` or `APP_ADMIN_USERS`
3. Assigns appropriate roles automatically
4. All other users get USER role by default

### Authentication Flow
1. User logs in with credentials
2. Backend validates credentials
3. Backend generates JWT token with user info and roles
4. Frontend stores token (localStorage or cookie)
5. Frontend sends token with each API request
6. Backend validates token and checks roles

### Protected Endpoints
In your code, you can protect endpoints by role:
```java
// Spring Boot example
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/api/admin/users")
public ResponseEntity<List<User>> getUsers() { ... }
```

```python
# Django example
@permission_required('rbac.admin')
def get_users(request): ...
```

---

## Common Issues

### "Invalid Token"
**Problem**: JWT token is rejected.

**Solutions**:
- Check `JWT_SECRET_KEY` matches between environments
- Verify token hasn't expired
- Ensure token is sent in `Authorization: Bearer <token>` header
- Check clock sync on servers (JWT validates timestamps)

### "User Has No Roles"
**Problem**: User logged in but has no permissions.

**Solutions**:
- Verify database connection is working
- Check `APP_ROOT_USERS` and `APP_ADMIN_USERS` are set correctly
- Ensure user email matches exactly (case-sensitive)
- Try logging out and logging in again

### "Access Denied"
**Problem**: User can't access an endpoint despite being logged in.

**Solutions**:
- Check user's actual roles in database
- Verify endpoint's required role matches user's role
- Ensure role hierarchy is configured correctly (ROOT > ADMIN > USER)
- Check if extension-auth0 is also enabled (might conflict)

---

## Best Practices

1. **Secret Management**:
   - Never commit `JWT_SECRET_KEY` to code
   - Use different secrets for each environment
   - Rotate secrets periodically (every 90 days)

2. **Root Users**:
   - Minimize ROOT users (1-3 maximum)
   - Use personal emails, not shared accounts
   - Enable MFA/2FA for ROOT users

3. **Token Expiration**:
   - Shorter is more secure
   - Balance security vs user experience
   - Consider refresh tokens for longer sessions

4. **Role Assignment**:
   - Start conservative (fewer admins)
   - Add more admins as needed
   - Regularly audit who has elevated roles

---

## Additional Resources

- **JWT Introduction**: https://jwt.io/introduction
- **Spring Security**: https://spring.io/guides/topicals/spring-security-architecture
- **Django REST Framework Auth**: https://www.django-rest-framework.org/api-guide/authentication/
