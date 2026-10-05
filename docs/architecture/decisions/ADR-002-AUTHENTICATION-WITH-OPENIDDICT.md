# ADR-002 — Authentication with OpenIddict

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Authentication / Security  
**Scope:** Backend, Frontend, API, Token Lifecycle, User Identity  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`

---

# 1. Context

Shinera requires authentication for multiple user types:

```text
Owner
Admin
Receptionist
Staff / Specialist
Customer
```

The primary interactive application is an Angular browser application communicating with the Shinera ASP.NET Core API.

Authentication must support:

```text
Login
Logout
Access Token
Refresh Token
Token Rotation
Token Revocation
Current User
Workspace Selection
Future External Clients
Future Social/Federated Login
```

Shinera already uses a custom business User model and intentionally does not use ASP.NET Core Identity as the core membership system.

The authentication protocol layer must therefore integrate with Shinera's own user/membership model instead of dictating it.

---

# 2. Decision

Shinera will use:

```text
OpenIddict
```

as its OAuth 2.0 / OpenID Connect authorization server and token infrastructure.

The first-party Angular application will use:

```text
Authorization Code Flow
+
PKCE
+
Refresh Token Flow
```

The Angular application is treated as a:

```text
Public Client
```

and therefore must not contain a client secret.

The Resource Owner Password Credentials grant ("Password Flow") will not be used for the main Shinera application.

The Implicit Flow will not be used.

---

# 3. High-Level Architecture

```text
Angular SPA
    │
    │ Authorization Code + PKCE
    ▼
Shinera Backend
    │
    ├── OpenIddict Server
    ├── Shinera User Authentication
    ├── Token Issuance
    ├── Token Validation
    ├── Tenant Membership
    └── Authorization
    │
    ▼
Shinera API
```

For the MVP, the Authorization Server and Resource/API Server live inside the same backend application.

This keeps deployment and validation simple while preserving standards-based authentication.

---

# 4. Why OpenIddict

OpenIddict is selected because it provides protocol-level support for:

```text
OAuth 2.0
OpenID Connect
Authorization Code
PKCE
Refresh Tokens
Token Revocation
Token Storage
Application Registration
Scopes
Claims
ASP.NET Core integration
EF Core integration
```

while allowing Shinera to retain its own:

```text
User model
Password verification
Tenant membership
Permission model
Business identity rules
```

OpenIddict does not require ASP.NET Core Identity.

This matches Shinera's architecture.

---

# 5. Separation of Responsibilities

OpenIddict is responsible for:

```text
OAuth/OIDC protocol
Authorization requests
Authorization codes
Token issuance
Token lifecycle
Refresh tokens
Application/client registrations
Token validation
Protocol security
```

Shinera is responsible for:

```text
User records
Password hashes
User status
Tenant memberships
Branch memberships
Business roles
Permissions
Subscription
Workspace context
Login policy
```

Do not move Shinera business authorization into OpenIddict application/client permissions.

---

# 6. User Authentication vs OAuth Authorization

These are different concerns.

OpenIddict answers:

```text
How are OAuth/OIDC requests processed?
How are tokens issued and validated?
```

Shinera answers:

```text
Is this username/password valid?
Is this User active?
May this User enter the requested workspace?
```

The Shinera application authenticates the human user and creates the `ClaimsPrincipal` used by OpenIddict.

---

# 7. User Model

Shinera owns the User entity.

Conceptually:

```text
User
├── Id
├── FirstName
├── LastName
├── Phone
├── Email
├── PasswordHash
├── IsActive
├── LastLoginAt
└── ...
```

OpenIddict protocol entities are separate from the Shinera User entity.

Do not use an OpenIddict Application record as a User.

Do not couple User domain behavior to OpenIddict persistence entities.

---

# 8. Password Handling

Shinera stores:

```text
PasswordHash
```

never plaintext passwords.

Password hashing should use a well-established ASP.NET Core password hashing implementation or an equivalent secure password hashing abstraction.

Requirements:

```text
Salted password hash
No reversible encryption
No plaintext logging
No plaintext storage
Constant-time verification through trusted library
Upgrade path for hashing parameters
```

Password verification belongs to Shinera authentication services.

---

# 9. Selected OAuth/OIDC Flow

The primary browser flow is:

```text
Authorization Code
+
PKCE
```

Conceptually:

```text
Angular
  ↓
Generate code_verifier
  ↓
Generate code_challenge
  ↓
/connect/authorize
  ↓
User Authentication
  ↓
Authorization Code
  ↓
Angular callback
  ↓
/connect/token
  + code
  + code_verifier
  ↓
Access Token
Refresh Token
```

PKCE is mandatory for the Angular client.

---

# 10. Why Password Grant Is Rejected

The following flow is rejected for the primary application:

```text
Angular
  ↓ username/password
/connect/token
  ↓
grant_type=password
```

Reasons:

- Password Grant is not recommended for new applications.
- It tightly couples the client to direct credential collection for token exchange.
- It is less suitable for future MFA/federation.
- It weakens the separation between interactive authentication and token issuance.
- Authorization Code + PKCE is the preferred browser-client architecture.

OpenIddict support for a grant does not mean Shinera must enable it.

---

# 11. Why Implicit Flow Is Rejected

Implicit Flow is not used.

Shinera does not issue access tokens directly from the authorization endpoint to the browser.

Authorization Code + PKCE provides the intended interactive browser flow.

---

# 12. Angular Client Type

The Angular application is registered as a:

```text
Public OpenIddict Client
```

It has:

```text
ClientId
Redirect URIs
Post-logout Redirect URIs
Allowed Endpoints
Allowed Grant Types
Allowed Response Types
Allowed Scopes
PKCE Requirement
```

It does not have:

```text
ClientSecret
```

A secret embedded in a browser bundle is not a secret.

---

# 13. OpenIddict Application Registration

Conceptual Angular client:

```text
ClientId: shinera-web

Type:
Public

Grant Types:
Authorization Code
Refresh Token

Response Type:
Code

Requirements:
PKCE

Scopes:
openid
profile
email
offline_access
shinera_api
```

Exact client identifiers and URIs are environment configuration.

---

# 14. Redirect URI Rules

Redirect URIs must be exact and environment-specific.

Examples:

```text
Development:
https://localhost:<port>/auth/callback

Production:
https://app.shinera.../auth/callback
```

Avoid broad wildcard redirect URIs.

Allowed origins and redirect URIs must be explicitly configured.

---

# 15. Authorization Server Endpoints

Initial protocol endpoints:

```text
/connect/authorize
/connect/token
/connect/logout
```

Additional endpoints may be enabled only when needed.

Potential future endpoints:

```text
/connect/revocation
/connect/userinfo
/connect/introspect
```

Do not enable protocol surface merely because OpenIddict supports it.

---

# 16. Login UX

Shinera may keep its own Angular login experience.

OpenIddict does not require Shinera to use ASP.NET Core Identity UI.

The interactive authentication process must still establish a trusted server-side authenticated principal for the authorization request.

Conceptually:

```text
Authorization request
↓
No authenticated Shinera session
↓
Shinera login UI
↓
Credentials submitted to trusted backend
↓
Verify User
↓
Establish interactive authentication state
↓
Resume authorization request
↓
Issue authorization code
```

The exact endpoint/UI handoff may follow the implementation conventions in the backend/frontend repositories.

---

# 17. Interactive Authentication Cookie

If a server-side cookie is used to maintain the interactive authorization session, it is separate from API bearer authorization.

The cookie should be:

```text
HttpOnly
Secure
Appropriate SameSite policy
Short/reasonable lifetime
```

Cookie-authenticated state-changing endpoints must consider CSRF protection.

The existence of an authorization/login cookie does not replace access-token validation on API calls.

---

# 18. Token Types

Shinera uses:

```text
Access Token
Refresh Token
```

and may receive an:

```text
ID Token
```

as part of OpenID Connect where required by the selected scopes/client behavior.

---

# 19. Access Token

Access Tokens are:

```text
Short-lived
Bearer credentials
Issued for Shinera API
Validated by backend
```

Initial target lifetime:

```text
15 minutes
```

This value is configuration and may be tuned without changing the architecture.

Short access-token lifetime limits the impact of token theft and delayed revocation.

---

# 20. Refresh Token

Refresh Tokens are used to obtain new Access Tokens without requiring the user to repeatedly authenticate.

Initial target lifetime:

```text
30 days
```

with:

```text
Rolling refresh tokens enabled
Sliding expiration enabled
```

unless later security/product requirements change.

Exact lifetime is configuration.

---

# 21. Refresh Token Rotation

Rolling refresh tokens remain enabled.

Conceptually:

```text
Refresh Token A
↓ use
New Access Token
New Refresh Token B
↓
Refresh Token A becomes redeemed
```

The frontend must replace the stored/current refresh token after every successful refresh.

Do not deliberately reuse an old refresh token.

---

# 22. Concurrent Refresh Requests

Angular must prevent multiple concurrent refresh operations.

Problem:

```text
Request A → 401
Request B → 401
Request C → 401
```

Bad behavior:

```text
A refreshes token
B refreshes same old token
C refreshes same old token
```

Preferred behavior:

```text
First failed request
↓
Start one refresh operation
↓
Other requests wait
↓
Receive new token set
↓
Replay waiting requests
```

The frontend authentication service/interceptor must implement a single-flight refresh mechanism.

---

# 23. Access Token Storage

Access tokens should not be persisted longer than required.

Preferred browser behavior:

```text
Access Token → in-memory application state
```

Avoid treating:

```text
localStorage
sessionStorage
IndexedDB
```

as secure secret stores.

Browser-accessible storage remains exposed to successful XSS.

---

# 24. Refresh Token Browser Storage

Long-lived refresh tokens require stricter treatment than access tokens.

The project must not assume that `localStorage` is a secure location for refresh tokens.

For the direct-SPA architecture, the implementation must minimize browser persistence and account for XSS risk.

Before introducing persistent long-lived browser refresh-token storage, evaluate whether Shinera should move the first-party web client to a BFF / HttpOnly-cookie model.

This ADR does not require a BFF for MVP, but it explicitly rejects casual long-lived refresh-token storage in `localStorage`.

---

# 25. Future BFF Option

If browser token exposure becomes unacceptable or security requirements increase, Shinera may introduce a Backend-for-Frontend architecture.

Conceptually:

```text
Angular
  ↓ Secure HttpOnly session cookie
BFF
  ↓ OAuth/OIDC tokens
Shinera API
```

In that model the browser never directly handles refresh tokens.

Moving to BFF is a future architectural decision and would require its own ADR/update.

---

# 26. Token Format

For the current same-backend Authorization Server + API architecture, Shinera will retain OpenIddict's secure default token protection behavior unless interoperability requires otherwise.

Authorization codes and refresh tokens remain protected/encrypted.

Access-token encryption should not be disabled merely to make tokens readable in Angular.

Angular should obtain application/user data through APIs such as:

```text
/auth/me
```

instead of decoding access-token internals as application state.

---

# 27. JWT Claim Reading in Frontend

Frontend business behavior must not depend on manually decoding access-token claims.

Bad:

```text
Decode token
→ infer permissions
→ infer subscription
→ infer tenant membership
```

Preferred:

```text
/auth/me
→ User
→ Active workspace
→ Permissions
→ Features
```

Tokens are credentials, not the frontend's canonical application state.

---

# 28. Token Storage in OpenIddict

OpenIddict token storage remains enabled.

Do not call:

```text
DisableTokenStorage()
```

for the normal Shinera server.

Token storage supports:

```text
Revocation
Refresh-token lifecycle
Security tracking
Future session management
```

Authorization storage also remains enabled unless a separate ADR changes this.

---

# 29. Reference Tokens

Reference access/refresh tokens are not required for the MVP by default.

The current architecture may use OpenIddict's normal protected token format plus token database entries.

Reference tokens may be considered later if:

```text
token size
centralized validation
storage requirements
security model
```

justify them.

Enabling reference tokens requires careful protection of stored payloads.

---

# 30. API Token Validation

Because OpenIddict Server and Shinera API are initially hosted in the same backend application, API validation should use OpenIddict's local-server integration.

Conceptually:

```text
OpenIddict Validation
+
UseLocalServer()
+
ASP.NET Core authentication
```

This avoids an unnecessary network introspection call for every API request.

---

# 31. Immediate Revocation vs Performance

By default, a validated short-lived access token may remain usable until expiration even after its refresh token/session is revoked.

MVP decision:

```text
Short access-token lifetime
+
Refresh-token revocation
```

is the default balance.

Do not add a database lookup on every authenticated API request only for theoretical immediate revocation.

If product/security requirements later demand immediate access-token invalidation, evaluate:

```text
Token entry validation
Authorization entry validation
Session/version validation
Introspection
```

and document the performance/security tradeoff.

---

# 32. Logout

Logout must:

```text
End interactive authentication session
Revoke the relevant refresh-token/session chain where possible
Clear frontend authentication state
Redirect safely to an allowed post-logout URI
```

The frontend must not treat deletion of a local token value as full server logout.

---

# 33. Logout Security Semantics

After logout:

```text
Refresh Token
→ must no longer be usable
```

A previously issued short-lived access token may remain valid until expiration unless immediate token-entry validation is enabled.

This is an intentional architecture tradeoff.

---

# 34. User Deactivation

If:

```text
User.IsActive == false
```

new login/authorization must fail.

Refresh handling must also revalidate security-sensitive user state before issuing a new token.

A disabled user must not be able to continue extending a session indefinitely through refresh tokens.

---

# 35. Tenant Deactivation

Authentication and Tenant access are separate.

A valid User token does not guarantee that a Tenant remains usable.

For every workspace:

```text
User authenticated
+
Tenant membership valid
+
Tenant active
```

must be satisfied.

A deactivated Tenant must fail workspace operations even if the User's access token is otherwise valid.

---

# 36. Authentication Is User-Centric

Access Tokens identify the authenticated user/client.

Tenant access is not granted merely because a token contains a TenantId claim.

This follows `ADR-001`.

Shinera must validate the selected workspace against current membership.

---

# 37. Tenant Selection and Access Token

A User may belong to multiple Tenants.

Therefore the primary access token should not permanently bind the User to exactly one Tenant in a way that requires reauthentication for every workspace switch.

Conceptually:

```text
Access Token
→ Who is the User?

Workspace Selector
→ Which Tenant/Branch is requested?

Backend Validation
→ Is the User allowed there?
```

---

# 38. Workspace Selector

The frontend may send a selected Tenant/Branch identifier using the project's workspace transport convention.

Possible transport:

```text
X-Tenant-Id
X-Branch-Id
```

or an equivalent server-approved mechanism.

These values are selectors, not trusted authorization claims.

Backend validates:

```text
User membership
Tenant state
Branch relationship
Branch access
```

before building `ICurrentTenant`.

---

# 39. Authentication Claims

Keep access-token claims minimal and stable.

Potential claims:

```text
sub
name
email
client_id / presenter
session identifier where needed
```

Do not put the entire dynamic Shinera permission catalog into long-lived token claims.

Do not put large subscription feature lists into tokens.

---

# 40. Permission Claims

Permissions may change while a session is active.

Therefore Shinera authorization should resolve current permission state through its authorization subsystem rather than treating token permission claims as permanently authoritative.

Possible optimization:

```text
Server-side permission cache
```

with safe invalidation.

Detailed design belongs to:

```text
ADR-003 — Authorization and Permission Model
```

---

# 41. Role Claims

Roles may be included only if useful for identity/display or compatibility.

Business authorization must still use permissions.

Avoid:

```text
[Authorize(Roles = "Owner")]
```

for rules that are truly permission-based.

---

# 42. Scope Design

Initial protocol/application scopes:

```text
openid
profile
email
offline_access
shinera_api
```

Additional scopes should represent API/resource access, not individual UI buttons or business permissions.

OAuth scope is not a replacement for Shinera's internal permission model.

---

# 43. Client Application Permissions

OpenIddict client registrations must explicitly receive only the endpoints, grant types, response types, and scopes they require.

For Angular:

```text
Authorization Endpoint
Token Endpoint
End Session Endpoint

Authorization Code
Refresh Token

Response Type: Code

Required API scopes
```

Do not globally ignore application permission checks without a clear reason.

---

# 44. Scalar / Development Clients

Development tools such as:

```text
Scalar
Postman
Integration test clients
```

must use separate client registrations where interactive authentication is needed.

Do not reuse the production Angular client secretlessly and indiscriminately for every tool.

Each development client should receive only the permissions it requires.

---

# 45. Machine-to-Machine Authentication

Future machine clients may use:

```text
Client Credentials
```

when required.

This is not used for human user login.

Machine client authorization must be separately registered and scoped.

---

# 46. Social / External Login

Future external authentication may include:

```text
Google
Apple
Microsoft
other identity providers
```

The architecture supports this because interactive user authentication is separate from OpenIddict token issuance.

An external identity can eventually be linked to a Shinera User.

This is outside MVP.

---

# 47. MFA

Multi-factor authentication is not required for MVP.

The selected Authorization Code architecture leaves room for MFA without changing the OAuth client flow.

If MFA is introduced:

```text
Authenticate primary factor
↓
MFA challenge
↓
Establish authenticated principal
↓
Continue authorization request
```

---

# 48. Password Reset

Password reset is an application authentication concern.

Future flow:

```text
Request reset
↓
Generate short-lived one-time reset token
↓
Verify token
↓
Set new password hash
↓
Optionally revoke existing sessions
```

Password reset tokens must not be ordinary OpenIddict access tokens.

---

# 49. Brute-Force Protection

Login endpoints require:

```text
Rate limiting
Failed-attempt monitoring
Generic invalid-credential responses
```

Future lockout policy may be introduced if product/security needs require it.

Do not reveal whether an email/phone exists unnecessarily.

---

# 50. Captcha

Captcha may be applied to:

```text
Login
Registration
Password reset
```

based on current product/security requirements.

Captcha complements but does not replace:

```text
Rate limiting
Credential security
Server-side validation
```

---

# 51. HTTPS

Production OAuth/OIDC traffic requires HTTPS.

Do not disable transport security requirements in production.

Development-only exceptions must remain environment-gated.

---

# 52. CORS

CORS must explicitly allow only approved frontend origins.

Avoid:

```text
AllowAnyOrigin
```

for authenticated production APIs.

Allowed origins are environment configuration.

CORS is not an authentication mechanism.

---

# 53. CSRF

Bearer-token API requests are not automatically subject to the same CSRF model as cookie-authenticated API requests.

However, any cookie-based authentication/session endpoint must consider CSRF.

If a future BFF uses cookies for API authentication, anti-CSRF protection becomes mandatory for state-changing requests.

---

# 54. XSS

The Angular application must treat XSS as an authentication security threat.

Because a successful XSS can access JavaScript-visible tokens/state:

```text
Avoid unsafe HTML injection
Avoid bypassSecurityTrust... without review
Use Angular sanitization
Apply appropriate CSP where practical
Avoid insecure token persistence
```

Authentication design cannot fully compensate for an XSS-compromised browser client.

---

# 55. Signing and Encryption Credentials

Development may use OpenIddict development certificates.

Production must use durable signing/encryption credentials.

Production must not use ephemeral keys.

Requirements:

```text
Persistent certificates/keys
Secure secret/key storage
Rotation capability
Environment-specific configuration
```

Signing and encryption credentials should be managed deliberately.

---

# 56. Key Rotation

Architecture must allow overlapping signing/encryption credentials during rotation.

Rotation must not unexpectedly invalidate all active tokens unless intentionally planned.

Operational key-rotation instructions may be documented later.

---

# 57. Claims Destinations

Only claims required by the recipient should be placed in:

```text
Access Tokens
ID Tokens
```

Sensitive internal claims must not be exposed merely because they exist on the server-side principal.

Claims used only for authorization codes/refresh processing can remain internal/protected.

---

# 58. `/auth/me`

Shinera provides an application-specific current-user endpoint such as:

```text
GET /auth/me
```

Purpose:

```text
User profile
Available Tenant memberships
Current workspace
Available Branches
Effective permissions
Effective feature entitlements
```

Exact response may be split into multiple endpoints as implementation evolves.

This endpoint is application state, not an OAuth protocol endpoint.

---

# 59. `/auth/me` and Token Claims

Frontend should prefer:

```text
/auth/me
```

over decoding token payloads for:

```text
Role
Permission
Feature
Tenant
Branch
User display data
```

This keeps authorization state current and avoids client coupling to token internals.

---

# 60. Authentication Error Semantics

Use:

```text
401 Unauthorized
```

when authentication credentials are:

```text
missing
invalid
expired
revoked where validation detects it
```

Use:

```text
403 Forbidden
```

when:

```text
User is authenticated
but lacks permission/access to perform the operation
```

Do not return 401 for ordinary authorization failure merely to trigger token refresh.

---

# 61. Refresh Interceptor Rule

Frontend must refresh only for authentication-expiration conditions.

It must not blindly refresh on every:

```text
401
403
business error
```

A refresh retry loop must be prevented.

Conceptual:

```text
API request
↓
Authentication-expired response
↓
Has refresh capability?
    ↓ yes
Single refresh
    ↓
Retry original request once
```

If refresh fails:

```text
Clear auth state
→ return to login
```

---

# 62. Refresh Failure

Refresh should fail when:

```text
Refresh token expired
Refresh token revoked
Refresh token already invalid/redeemed outside allowed behavior
User inactive
Client invalid
Authorization invalid
```

Frontend must not keep retrying refresh indefinitely.

---

# 63. Session Management

A Shinera session concept may be built on top of OpenIddict authorization/token records.

Potential future capabilities:

```text
List active sessions
Revoke current session
Revoke other sessions
Revoke all user sessions
Single-session policy
Device information
```

These are application-level session features and may require additional metadata beyond OpenIddict's protocol entities.

---

# 64. Single Active Session

Shinera does not require a one-user/one-session restriction as a universal architecture rule unless product requirements explicitly add it.

If introduced later:

- implement it as session policy,
- revoke the old session/authorization explicitly,
- do not misuse 401/refresh behavior to emulate it,
- add dedicated error/session semantics.

---

# 65. Token Revocation Strategy

Revocation targets may include:

```text
Refresh token
Authorization chain
User sessions
Client grant
```

The exact operation depends on the use case.

Logout should at minimum prevent the current refresh path from extending the session.

Security/admin actions may revoke broader authorization state.

---

# 66. Authorization Storage

OpenIddict authorization storage remains enabled.

Reasons:

```text
Token chain tracking
Revocation
Security
Future session management
```

Disabling authorization storage is rejected for the normal Shinera authentication server.

---

# 67. Database Integration

OpenIddict uses EF Core persistence integrated into Shinera backend infrastructure.

Protocol tables/entities remain infrastructure concerns.

Shinera business entities should not directly depend on OpenIddict database types.

---

# 68. Custom OpenIddict Application Entity

Shinera may extend OpenIddict application entities with metadata such as:

```text
ProductKey
Description
IsActive
```

when there is a real platform requirement.

Rules:

- preserve required OpenIddict mappings,
- avoid duplicate/conflicting EF configuration,
- keep protocol behavior compatible,
- do not put unrelated business state into OpenIddict tables.

Custom mapping changes require integration tests and migration review.

---

# 69. OpenIddict DbContext Configuration

If custom OpenIddict entities are used:

```text
One authoritative EF mapping configuration
```

must exist for each entity.

Avoid:

```text
Default OpenIddict mapping
+
Conflicting custom mapping
```

that defines the same property/relationship differently.

OpenIddict model customization must be verified against generated migrations.

---

# 70. Authentication Database Migration

Authentication schema changes must be reviewed carefully because they may affect:

```text
Applications
Authorizations
Scopes
Tokens
Existing sessions
Client registrations
```

Migration rollback/upgrade behavior should be understood before production deployment.

---

# 71. OpenIddict Versioning

Use a supported OpenIddict release compatible with the current .NET runtime.

At the time of this ADR, current OpenIddict documentation explicitly supports ASP.NET Core/.NET 10.

Package upgrades must review OpenIddict migration guides before adoption.

Do not perform major-version upgrades casually.

---

# 72. Development Credentials

Local development may use:

```text
AddDevelopmentSigningCertificate()
AddDevelopmentEncryptionCertificate()
```

or equivalent development configuration.

Do not copy development certificate configuration blindly into production.

---

# 73. Production Credentials

Production configuration must load durable credentials from an appropriate secure source.

Examples:

```text
Certificate store
Secret manager
Mounted protected certificate
Cloud key management
```

Exact deployment mechanism is environment-specific.

---

# 74. Authentication Logging

Authentication logs may include:

```text
UserId
ClientId
Grant Type
Result
TraceId
Timestamp
```

where appropriate.

Do not log:

```text
Password
Access Token
Refresh Token
Authorization Code
Client Secret
Raw authentication payload
```

---

# 75. Audit Events

Security-relevant events should be auditable where practical:

```text
Login success
Login failure
Logout
Password change
Password reset
Session revocation
User disabled
Important authentication configuration changes
```

Audit data must not contain credentials.

---

# 76. Authentication Tests

Required integration coverage should include:

```text
Valid login/authorization
Invalid credentials
Inactive user
Authorization code exchange
Invalid code verifier
Refresh success
Refresh rotation
Old refresh token rejection
Refresh after user disable
Logout/revocation
Expired access token
Unauthorized API access
Malformed token
Invalid client
Invalid redirect URI
```

---

# 77. PKCE Tests

Integration tests must verify:

```text
Valid code_verifier
→ token issued
```

and:

```text
Missing/wrong code_verifier
→ token rejected
```

for the Angular public client.

---

# 78. Refresh Concurrency Tests

Frontend tests should verify:

```text
multiple simultaneous expired API calls
→ one refresh request
→ waiting requests reuse new access token
```

Backend integration tests should verify rolling refresh-token behavior according to configured OpenIddict semantics.

---

# 79. Tenant Integration Tests

Authentication tests must interact correctly with `ADR-001`.

Examples:

```text
Authenticated User without Tenant membership
→ cannot enter Tenant

Authenticated User with membership
→ may select Tenant

Changing X-Tenant-Id to unrelated Tenant
→ denied

Valid token
+
inactive Tenant
→ workspace operation denied
```

---

# 80. Permission Integration

A valid access token proves identity/client authentication.

It does not prove every business permission.

Flow:

```text
Token valid
↓
Current User
↓
Tenant membership
↓
Branch access
↓
Permission evaluation
↓
Feature entitlement
↓
Operation
```

Detailed permission design belongs to ADR-003.

---

# 81. Authentication Flow Summary

Primary Angular flow:

```text
Angular SPA
↓
Authorization request
↓
OpenIddict /connect/authorize
↓
Shinera interactive authentication
↓
Authorization code
↓
PKCE token exchange
↓
Access token + refresh token
↓
API calls
↓
Access token expires
↓
Single-flight refresh
↓
New access token + rolling refresh token
```

---

# 82. Current User / Workspace Flow

```text
Valid access token
↓
GET /auth/me
↓
User identity
↓
Tenant memberships
↓
Select workspace
↓
Backend validates membership
↓
ICurrentTenant
↓
Authorized application requests
```

---

# 83. Logout Flow

```text
User clicks Logout
↓
Revoke current refresh/session authorization
↓
End interactive auth session
↓
Clear frontend auth state
↓
OpenIddict end-session flow where applicable
↓
Redirect to approved URL
```

---

# 84. Rejected Alternatives

## ASP.NET Core Identity as Core User System

Rejected as an architectural requirement.

Reason:

```text
Shinera owns its User/membership model.
OpenIddict does not require ASP.NET Identity.
```

Individual ASP.NET security components may still be used where useful.

---

## Password Grant for Angular

Rejected.

Reason:

```text
Legacy/non-recommended flow for new browser apps
Poor future fit for MFA/federation
```

---

## Implicit Flow

Rejected.

---

## Client Secret in Angular

Rejected.

A browser application cannot securely keep a client secret.

---

## Long-Lived Access Tokens

Rejected.

Use short-lived access tokens + refresh mechanism.

---

## Permissions as Permanent Token Claims

Rejected as the primary authorization source.

Permissions are dynamic business state.

---

## localStorage as "Secure Token Vault"

Rejected.

Browser local storage is accessible to JavaScript and therefore exposed to successful XSS.

---

# 85. Consequences

## Positive

This decision provides:

```text
Standards-based authentication
Modern browser flow
PKCE protection
Future MFA compatibility
Future social login compatibility
Refresh-token support
Separation from business authorization
No dependency on ASP.NET Identity
OpenIddict revocation/storage support
```

## Negative

It introduces:

```text
Redirect-based authorization flow
More frontend auth-state complexity
Refresh-token lifecycle complexity
Need for single-flight refresh handling
Need for secure browser-token strategy
Additional OpenIddict persistence tables
```

These costs are accepted.

---

# 86. Security Tradeoff

Direct SPA token handling is simpler operationally than BFF but exposes JavaScript-visible credentials to XSS risk.

For MVP:

```text
Authorization Code + PKCE direct SPA
```

is accepted, with strict token-handling rules.

If security requirements increase:

```text
BFF / HttpOnly-cookie architecture
```

should be reevaluated.

---

# 87. Follow-Up ADRs

This ADR directly interacts with:

```text
ADR-001 — Multi-Tenancy Strategy
ADR-003 — Authorization and Permission Model
ADR-006 — Subscription and Feature Gating
ADR-007 — Staff vs User Identity Model
```

Potential future ADR:

```text
ADR-009 — Browser Token Storage / BFF Strategy
```

only if the implementation requires a deeper decision.

---

# 88. Definition of Done Integration

Authentication stories are not Done unless applicable checks include:

```text
Protocol flow verified
PKCE verified
Client permissions verified
Token lifetime configured
Refresh behavior verified
Logout/revocation verified
Inactive user behavior verified
No secrets in frontend
No credential/token logging
HTTPS production configuration
Relevant integration tests
Frontend refresh race handling
```

---

# 89. Final Decision Summary

Shinera authentication will use:

```text
OpenIddict
+
Custom Shinera User Model
+
Authorization Code Flow
+
Mandatory PKCE
+
Public Angular Client
+
Short-lived Access Tokens
+
Rolling Refresh Tokens
+
OpenIddict Token/Authorization Storage
+
Local API Token Validation
```

The primary Angular application will not use:

```text
Password Grant
Implicit Flow
Client Secret
ASP.NET Identity as required User model
```

Authentication proves who the User is.

Tenant membership, Branch access, Permissions, and Feature entitlement remain separate server-side authorization concerns.

Tokens are treated as credentials, not as the canonical source of Shinera application state.

---

# 90. Official OpenIddict References

- Choosing the right flow: https://documentation.openiddict.com/guides/choosing-the-right-flow.html
- PKCE: https://documentation.openiddict.com/configuration/proof-key-for-code-exchange
- Application permissions: https://documentation.openiddict.com/configuration/application-permissions
- Token storage: https://documentation.openiddict.com/configuration/token-storage.html
- Token formats: https://documentation.openiddict.com/configuration/token-formats
- Authorization storage: https://documentation.openiddict.com/configuration/authorization-storage
- ASP.NET Core integration: https://documentation.openiddict.com/integrations/aspnet-core
- Signing/encryption credentials: https://documentation.openiddict.com/configuration/encryption-and-signing-credentials.html
