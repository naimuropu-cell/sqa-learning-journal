# OAuth 2.0 Token Revocation, Refresh Token Rotation & Session Expiry Testing

## 1. OAuth 2.0 / OIDC Session Security Overview

In modern web and mobile architectures, authentication and authorization rely on token-based protocols (**OAuth 2.0** and **OpenID Connect**). Typically:
- **Access Tokens (JWT)**: Short-lived credentials (e.g., 15 minutes) presented in the `Authorization: Bearer <token>` header to access protected API resources.
- **Refresh Tokens**: Long-lived credentials (e.g., 30 days) exchanged via the authorization server (`/oauth/token`) to issue new access tokens without requiring user re-authentication.

Because JWT access tokens are stateless and cryptographically verified by resource servers using public keys without hitting the database, **invalidating them immediately upon user logout or security revocation** poses a significant architectural and testing challenge.

```
┌─────────────────┐             ┌─────────────────┐             ┌─────────────────┐
│     Client      │             │  Auth Server    │             │ Resource Server │
└────────┬────────┘             └────────┬────────┘             └────────┬────────┘
         │                               │                               │
         │ 1. Token Refresh Request      │                               │
         │    (Presents Old Refresh Token│                               │
         ├──────────────────────────────►│                               │
         │                               │ 2. Issue NEW Access Token     │
         │                               │    & NEW Refresh Token        │
         │                               │    (Invalidate Old Token)     │
         │ 3. Return New Tokens          │                               │
         │◄──────────────────────────────┤                               │
         │                               │                               │
         │ 4. Access Protected API       │                               │
         │    (Authorization: Bearer)    │                               │
         ├──────────────────────────────────────────────────────────────►│
         │                               │                               │
         │ 5. Attacker Replays OLD Token │                               │
         │    (Replay Detected!)         │                               │
         ├──────────────────────────────►│                               │
         │                               │ 6. REVOKE ENTIRE FAMILY!      │
         │ 7. HTTP 401 Unauthorized      │    (Lock Account)             │
         │◄──────────────────────────────┤                               │
```

---

## 2. Testing Refresh Token Rotation (RTR) & Replay Attacks

In secure public clients (like SPAs and Mobile apps), RFC 6749 and OAuth 2.0 Security Best Current Practice (BCP) mandate **Refresh Token Rotation (RTR)**:
1. Every time a refresh token is used, the authorization server must invalidate it and issue a brand-new refresh token.
2. **Replay Detection & Family Invalidation**: If an already-used refresh token is presented again (indicating token theft or man-in-the-middle leakage), the authorization server must immediately invalidate the **entire family** of tokens issued to that session and force the user to log in again.

### Automated Test: Verifying Refresh Token Rotation and Replay Protection

Below is a Mocha test suite verifying single-use refresh tokens and token family revocation:

```javascript
const axios = require('axios');
const { expect } = require('chai');

const AUTH_URL = 'https://auth.staging.example.com';
const CLIENT_ID = 'qa-spa-client';

describe('OAuth 2.0 Refresh Token Rotation (RTR) & Security Suite', () => {
  let initialRefreshToken;
  let firstRotatedRefreshToken;
  let secondRotatedRefreshToken;

  before(async () => {
    // Initial login to acquire token pair
    const res = await axios.post(`${AUTH_URL}/oauth/token`, {
      grant_type: 'password',
      username: 'qa_security_user@example.com',
      password: 'TargetPassword123!',
      client_id: CLIENT_ID,
    });

    expect(res.status).to.equal(200);
    expect(res.data).to.have.property('access_token');
    expect(res.data).to.have.property('refresh_token');
    initialRefreshToken = res.data.refresh_token;
  });

  it('Step 1: Successfully rotate refresh token on valid refresh request', async () => {
    const res = await axios.post(`${AUTH_URL}/oauth/token`, {
      grant_type: 'refresh_token',
      refresh_token: initialRefreshToken,
      client_id: CLIENT_ID,
    });

    expect(res.status).to.equal(200);
    expect(res.data).to.have.property('access_token');
    expect(res.data).to.have.property('refresh_token');

    // Assert that the new refresh token is DIFFERENT from the old one
    expect(res.data.refresh_token).to.not.equal(initialRefreshToken);
    firstRotatedRefreshToken = res.data.refresh_token;
  });

  it('Step 2: Second valid rotation succeeds and produces third refresh token', async () => {
    const res = await axios.post(`${AUTH_URL}/oauth/token`, {
      grant_type: 'refresh_token',
      refresh_token: firstRotatedRefreshToken,
      client_id: CLIENT_ID,
    });

    expect(res.status).to.equal(200);
    secondRotatedRefreshToken = res.data.refresh_token;
    expect(secondRotatedRefreshToken).to.not.equal(firstRotatedRefreshToken);
  });

  it('Step 3: Replaying the first (already consumed) refresh token triggers family revocation', async () => {
    try {
      // Replay old token
      await axios.post(`${AUTH_URL}/oauth/token`, {
        grant_type: 'refresh_token',
        refresh_token: firstRotatedRefreshToken,
        client_id: CLIENT_ID,
      });
      expect.fail('Server should have rejected replayed refresh token');
    } catch (err) {
      expect(err.response.status).to.equal(400);
      expect(err.response.data.error).to.equal('invalid_grant');
    }

    // Crucial Security Verification: Even the latest legitimate token must now be invalid!
    try {
      await axios.post(`${AUTH_URL}/oauth/token`, {
        grant_type: 'refresh_token',
        refresh_token: secondRotatedRefreshToken,
        client_id: CLIENT_ID,
      });
      expect.fail('Latest token should have been revoked due to replay attack detection');
    } catch (err) {
      expect(err.response.status).to.equal(400);
      expect(err.response.data.error).to.equal('invalid_grant');
    }
  });
});
```

---

## 3. Token Revocation (RFC 7009) & Distributed Blacklisting

Under **RFC 7009**, authorization servers expose an endpoint (`/oauth/revoke`) where clients explicitly invalidate tokens on logout.

### QA Verification Matrix for Token Revocation:
1. **Revocation Response**: The endpoint must return `200 OK` whether the token is valid, already expired, or non-existent (to prevent token enumeration).
2. **Access Token Blacklist**: Since JWTs cannot be altered once signed, the resource server or API Gateway must query a low-latency distributed store (e.g., Redis `SETEX` with remaining JWT TTL) to block revoked JWT `jti` (JWT ID) claims.
3. **Subsequent API Invocations**: After calling `/oauth/revoke`, API requests using the old token must immediately return `401 Unauthorized`.

```bash
# Test token revocation via cURL
curl -X POST https://auth.staging.example.com/oauth/revoke \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token=eyJh...&token_type_hint=access_token&client_id=qa-spa-client"
```

---

## 4. Token Expiration & Clock Skew Edge Cases

QA test plans must validate:
- **`exp` (Expiration Time)**: Verify token is rejected precisely at $t = \text{exp} + \text{leeway}$.
- **Clock Skew Toleration**: In distributed cloud environments, servers may have slightly unsynchronized system clocks. Standard JWT libraries allow 10-60 seconds of clock skew. QA engineers test boundary timestamps:
  - Token timestamp $t - 5\text{s}$: Accepted within skew.
  - Token timestamp $t - 120\text{s}$: Rejected with `TokenExpiredError`.
- **`nbf` (Not Before)**: Ensure tokens are not accepted before their designated activation time.

---

## 5. QA Security Checklist for OAuth 2.0 Implementations

- [ ] **Single-Use Refresh Tokens**: Verify that using a refresh token immediately invalidates it.
- [ ] **Replay Detection**: Verify that re-using an expired/consumed refresh token invalidates the entire token session family.
- [ ] **RFC 7009 Revocation**: Ensure `/oauth/revoke` cleans up sessions and propagates revocation to edge proxies/Redis blacklists.
- [ ] **Audience (`aud`) & Issuer (`iss`) Validation**: Confirm that tokens issued for staging cannot be used in production or across tenant boundaries.
- [ ] **Secure Storage**: Confirm tokens are stored in `HttpOnly; Secure; SameSite=Strict` cookies or OS-level secure keystores (Keychain / Keystore), not unencrypted `localStorage`.
