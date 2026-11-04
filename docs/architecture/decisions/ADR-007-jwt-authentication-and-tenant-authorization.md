# ADR-007: JWT Authentication and Tenant Authorization

## Status
Accepted

## Date
2026-11-04

## Context

StockFlow is a multi-tenant SaaS platform where users from different companies access shared application infrastructure.

A user may belong to multiple companies and may have different roles in each company.

For example:

- A user may be an Inventory Manager in Company A.
- The same user may be an Order Manager in Company B.
- A user must not access another company's data without valid membership and authorization.

StockFlow needs a secure authentication and authorization strategy that supports:

- User authentication.
- Access token management.
- Refresh token management.
- Multiple tenant memberships.
- Active tenant selection.
- Role-based authorization.
- Secure tenant context propagation between microservices.
- Token expiration and revocation.

Authentication and authorization must be handled consistently across all services.

## Decision

StockFlow will use JWT-based authentication with short-lived access tokens and refresh tokens.

The Auth Service owns user identity, credentials, authentication, and token issuance.

The Tenant Service owns companies, memberships, and tenant-specific roles.

Authorization will be enforced at the service level. The API Gateway provides an initial security boundary but is not the only authorization layer.

The system will use Role-Based Access Control (RBAC) for the MVP.

## 1. Authentication and Authorization

Authentication and authorization are different responsibilities.

### Authentication

Authentication answers:

"Who is the user?"

Auth Service verifies user credentials and establishes the user's identity.

### Authorization

Authorization answers:

"Is this user allowed to perform this operation?"

Tenant Service owns tenant memberships and role assignments. Business services enforce the permissions required for their operations using trusted authorization context.

Example:

An Inventory Manager may be allowed to:

- View inventory.
- Receive stock.
- Adjust stock.
- View inventory movements.

But may not be allowed to:

- Manage user accounts.
- Change company settings.
- Modify system-wide permissions.

Authentication alone does not grant access to business operations.

## 2. Service Ownership

| Responsibility | Owning Service |
|---|---|
| User registration | Auth Service |
| Login and credential validation | Auth Service |
| Password hashing | Auth Service |
| Access token generation | Auth Service |
| Refresh token management | Auth Service |
| Company registration | Tenant Service |
| Company membership | Tenant Service |
| Tenant-specific roles | Tenant Service |
| Business permission enforcement | Respective business services |

The API Gateway routes requests and performs initial authentication checks.

It must not become the owner of user identity or tenant membership data.

## 3. JWT Access Tokens

After successful authentication and tenant selection, the client receives a short-lived access token.

The access token is a signed JWT containing claims such as:

| Claim | Purpose |
|---|---|
| sub | User identifier |
| iss | Token issuer |
| aud | Intended audience |
| iat | Issued-at timestamp |
| exp | Expiration timestamp |
| jti | Unique token identifier |
| tenant_id | Active tenant identifier |
| roles | Authorized tenant roles, if included |
| permissions | Permission claims, if included |

Example access token payload:

```json
{
  "sub": "user-uuid",
  "iss": "stockflow-auth",
  "aud": "stockflow-api",
  "iat": 1793786400,
  "exp": 1793787300,
  "jti": "token-uuid",
  "tenant_id": "tenant-uuid",
  "roles": ["INVENTORY_MANAGER"]
}
```

This is an illustrative payload, not a real token.

### Token security rules

- Access tokens must have a short expiration period.
- Tokens must be digitally signed.
- Services must validate the signature.
- Services must validate issuer and audience.
- Services must validate expiration and other required claims.
- The signing algorithm must be explicitly configured and allowlisted.
- Sensitive information must never be stored in JWT payloads.
- Passwords, password hashes, and refresh tokens must never appear in access tokens.

For production, StockFlow will prefer asymmetric signing such as RS256 or an appropriately configured EdDSA algorithm.

The Auth Service holds the private signing key. Other services receive the public verification key through a secure key-distribution mechanism.

Key rotation must be supported using key identifiers (`kid`) and overlapping verification keys during migration.

## 4. Refresh Tokens

Access tokens are short-lived. Refresh tokens allow users to obtain new access tokens without repeatedly entering their credentials.

Refresh tokens must be managed securely by Auth Service.

### Refresh token rules

1. Generate cryptographically secure random refresh tokens.
2. Store only a secure hash of each refresh token in the database.
3. Associate tokens with a user and session.
4. Configure an expiration time.
5. Rotate refresh tokens after successful use.
6. Revoke the previous token during rotation.
7. Detect reuse of an already-rotated token.
8. Revoke the affected token family or session when reuse is detected.
9. Support logout and session revocation.

Example Auth Service entities:

**RefreshToken**

- id
- user_id
- token_hash
- session_id
- expires_at
- revoked_at
- created_at

**UserSession**

- id
- user_id
- created_at
- last_used_at
- revoked_at

Refresh tokens must not be stored in plaintext.

For browser clients, refresh tokens should be handled using appropriately secured cookies, including HttpOnly, Secure, and suitable SameSite settings.

## 5. Multiple Tenant Memberships

A user may belong to multiple tenants.

For example:

| User | Tenant | Role |
|---|---|---|
| User A | Company A | OWNER |
| User A | Company B | INVENTORY_MANAGER |
| User B | Company A | ORDER_MANAGER |

Tenant Service owns these relationships.

A user must have an active membership before accessing a tenant's business data.

### Active tenant selection

The authentication flow will support selecting an active tenant after login.

Example flow:

1. User logs in.
2. Auth Service validates credentials.
3. The application retrieves the user's available tenant memberships through Tenant Service.
4. User selects a tenant.
5. Tenant membership and status are validated.
6. The application obtains an access token with the selected tenant context.
7. The client sends that token with subsequent requests.

The user may switch tenants by selecting another valid membership and obtaining a new tenant-scoped access token.

A tenant identifier supplied by the client is never sufficient proof of membership.

## 6. Role-Based Access Control (RBAC)

StockFlow will use RBAC for the MVP.

Roles define groups of permissions.

Initial roles include:

- PLATFORM_ADMIN
- OWNER
- COMPANY_ADMIN
- INVENTORY_MANAGER
- ORDER_MANAGER
- WAREHOUSE_OPERATOR

Tenant-specific roles are managed by Tenant Service.

Permissions should represent actions rather than only screen access.

Example permissions:

- `product.read`
- `product.create`
- `product.update`
- `warehouse.read`
- `warehouse.manage`
- `inventory.read`
- `inventory.receive`
- `inventory.adjust`
- `inventory.reserve`
- `order.read`
- `order.create`
- `order.cancel`
- `membership.manage`

Services must check the permissions required for each operation.

Role names alone should not become hardcoded authorization logic throughout the codebase.

Instead, define permission requirements and map roles to permissions through a controlled authorization policy.

Platform-level permissions must be kept separate from ordinary tenant-level permissions.

## 7. Tenant Context Propagation

Every business request must execute within a trusted tenant context.

The expected request flow is:

1. Client sends an access token to API Gateway.
2. Gateway validates the token.
3. Gateway forwards authenticated identity and tenant context to the target service through a trusted internal communication mechanism.
4. Target service validates the trusted identity context.
5. Target service verifies required permissions.
6. Target service applies tenant filtering to database operations.

Services must never trust arbitrary identity or tenant headers supplied directly by an untrusted client.

The implementation must prevent clients from overriding trusted internal headers.

Internal service-to-service communication must also be authenticated.

A service must not assume that a request is trusted merely because it originated from the API Gateway.

## 8. Authorization Responsibilities

Authorization must be enforced at multiple layers.

### API Gateway

- Validate incoming access tokens.
- Reject missing or invalid authentication.
- Apply rate limiting.
- Forward trusted identity context.
- Protect internal service routes from public access.

### Business Services

- Validate trusted authentication context.
- Check operation-level permissions.
- Validate tenant ownership.
- Apply tenant filters to database queries.
- Reject unauthorized cross-tenant operations.

The Gateway must not be the only security boundary.

If a business service is accessed through an internal route, it must still enforce its security requirements.

## 9. Token Expiration and Revocation

JWT access tokens are generally self-contained. This means that a service can validate a token without querying Auth Service on every request.

However, self-contained tokens cannot automatically reflect every account or membership change immediately.

StockFlow will use the following approach:

- Keep access tokens short-lived.
- Rotate refresh tokens.
- Revoke refresh tokens on logout.
- Revoke sessions when credentials or account status require it.
- Revalidate tenant membership and authorization for sensitive operations where necessary.
- Use an optional token revocation mechanism if immediate access-token invalidation becomes a requirement.

Revoking a refresh token does not automatically invalidate an already-issued access token.

The remaining validity period of an access token must be considered in the security design.

## 10. Alternatives Considered

### 1. Session-Based Authentication

Advantages:
- Centralized session management.
- Straightforward server-side session revocation.
- No need to embed authorization context in a signed token.

Disadvantages:
- Requires session storage and lookup.
- Adds server-side state.
- Distributed session management is required across services.

Not selected as the primary authentication approach for the MVP.

### 2. JWT Authentication

Advantages:
- Stateless access token verification.
- Efficient authentication across multiple services.
- Standardized token claims.
- Suitable for distributed systems.

Disadvantages:
- Revocation is more complicated.
- Signing key management is required.
- Claims can become stale.
- Token contents are signed, not encrypted.

Selected.

### 3. OAuth 2.0 / OpenID Connect Identity Provider

Advantages:
- Standardized authorization and identity protocols.
- External identity provider support.
- Mature identity and federation capabilities.

Disadvantages:
- Additional integration and configuration.
- May introduce operational dependencies.

Can be introduced later if StockFlow requires enterprise SSO or external identity federation.

## 11. Consequences

### Positive

- Consistent authentication across services.
- Supports users belonging to multiple tenants.
- Clear separation between identity and tenant membership.
- Short-lived access tokens reduce exposure.
- RBAC provides a manageable MVP authorization model.
- Services can independently enforce permissions.

### Negative

- Token lifecycle and key rotation require careful implementation.
- Tenant membership changes need a defined revocation strategy.
- Authorization logic must be consistent across services.
- Secure internal identity propagation requires additional infrastructure.

## 12. Implementation Notes

1. Implement user registration and login in Auth Service.
2. Use a modern password hashing algorithm such as Argon2id.
3. Implement signed access tokens.
4. Implement hashed refresh token storage and rotation.
5. Implement tenant membership and role management in Tenant Service.
6. Implement active tenant selection.
7. Define shared authentication and authorization contracts.
8. Configure secure public-key distribution and key rotation.
9. Add authentication guards and permission guards in NestJS.
10. Validate tenant ownership in every business service.
11. Protect internal service communication.
12. Add logout and session revocation.
13. Document authentication APIs using Swagger.

Shared packages may contain token claim types, permission identifiers, and validation contracts. They must not contain shared business entities or database repositories.

## 13. Testing Requirements

The implementation must test:

- Successful login.
- Invalid credentials.
- Expired access tokens.
- Invalid token signatures.
- Incorrect issuer and audience.
- Expired refresh tokens.
- Refresh token rotation.
- Refresh token reuse detection.
- Logout and session revocation.
- User belonging to multiple tenants.
- Switching between tenant contexts.
- Suspended tenant membership.
- Unauthorized role and permission access.
- Cross-tenant data access attempts.
- Forged tenant headers.
- Direct access to internal services without valid authentication.

## References

- [RFC 7519: JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8725: JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725)
- [RFC 9700: OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)