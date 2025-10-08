# Authentication Service

This service issues and validates tokens that prove the caller exists and what they are allowed to do. Tokens are JWTs and for MVP should at minimum support username/password authentication. To break it down the auth service handles:

- **Identity**: user/pass authentication with the option of MFA
- **Token Assurance**: access and refresh tokens with scopes in the `resource:method` format
- **Public Keys**: support JWT verification without additional API overhead
- **Sessions & Refresh**: Support rotation of keys and revoking of sessions
- **ACL Policies**: Use scopes and role based access controls (RBAC) though I'm not sure what of this will be MVP.

The auth service is a must need for the other services (media and chat). Later it would be good to have the auth service support multiple tenants.

## Endpoints

For the MVP of the authentication service, it should be able to handle simple user/password with JWT. This means we need the following endpoints:

- `POST /oauth/token` 
- `POST /oauth/revoke` 
- `GET /.well-known/jwks.json` *cacheable*
- `GET /v1/userinfo` 
- `POST /v1/invites/mint` *admin-only* generate invitation token
- `POST  /v1/signup?invite=<token>&srv=<server_id>` 
- `GET /healthz` Liveness → 200 OK
- `GET /readyz` Readiness when the DB access is ready
- `GET /.well-known` Publish limits and capabilities

These are the core endpoints for a functional MVP, but for the sake of "security best practises" we probably should have MFA support:

- `POST /v1/mfa/totp/enroll` 
- `POST /v1/mfa/totp/verify` 
- `POST /v1/mfa/backup-codes`  *Regenerate and invalidate codes*
- `DELETE /v1/mfa/totp`

> **Note**: Future development to support OpenID Connect (OIDC) we will need a `GET /.well-known/openid-config` 

These endpoints are only here for a functional MVP, but more endpoints will need to be added to help with user management, roles, scope and other admin controls.

## Login for Client (No MFA)

As mentioned earlier, the first version will only support username/password authentication `grant_type=password` to generate a JWT auth token. This can be validated using the `GET /.well-known/jwks.json` following RFC7517 to support `RS256`, `ES256` , and `EdDSA` following `RFC7518` .

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=password&
username=admin&
password=admin&
audience=<server_id>&
scope=rooms:read%20rooms:write
```

This is an example login request. The audience should be the chat service server ID, this will help us later for multi tenant support (if we ever can be arsed doing this). The response should be what you would expect for an OAuth response:

```json
{
  "access_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "...",
  "scope": "rooms:read rooms:write"
}
```

## Login for Client (with MFA)

Imagine being security conscience. Lets assume user `admin` from earlier logged in with that `grant_type=password` but they have MFA enabled. Instead of the server returning the access tokens as above, it will instead return the following **MFA Challenge:**

```json
{
  "mfa_required": true,
  "mfa_token": "...",
  "methods": ["totp", "backup_codes"]
}
```

The user will then make another request to the `POST /oauth/token` endpoint with the `grant_type=mfa_otp` and use the short lived `mfa_token` to link the authentication state of the user.

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=mfa_otp&
mfa_token=XXXXXXXXXXXXXXXXXXXXXXXXX&
method=totp&
otp_code=123456&
```

If you use the `method=backup_codes` you can use the single use backup codes for when user's lose their TOTP device. Though when you use a backup code it will remove them from being used again. A valid request with will return the access token response.

## Inviting New Members

From the previously mentioned endpoints, to invite a new member it will require the following:

1. Admin Generate an invite token with `POST /v1/invites/mint` with the `srv` and `role` set. Probably should have a flag which can set the invite token to be reusable and have an expiry time. The response will also include a link to an auth service signup page.
2. The authentication service should be able to support a signup page which can take the invite token and `srv` key from the previously generated API call.
3. The new member can login through the authentication service but they will require to call the `POST /v1/invites/redeem` endpoint on the Chat Service API to have their account activated.

> **Note**: If the user hasn't done a redeem request on the Chat Service API, then all chat service API endpoints will return with a `404 Forbidden` and contain a redeem hint location that needs to be called to activate the account.

## Well Known Data

Each service has a `GET /.well-known` endpoint which contains the common resources to make it easier to use and integrate with other systems. The following is a rough draft of the Authentication Service well known data:

```json
{
  "issuer": "https://auth.example.com",
  "token_endpoint": "https://auth.example.com/oauth/token",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "userinfo_endpoint": "https://auth.example.com/v1/userinfo",
  "mfa_supported": ["totp", "backup_codes"]
}
```

This data will have to be modified when we want to do OIDC integrations, and we probably don't need the MFA Supported field.