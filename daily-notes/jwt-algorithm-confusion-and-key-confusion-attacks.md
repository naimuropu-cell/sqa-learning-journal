# JWT Algorithm Confusion (RS256 vs HS256) & 'None' Algorithm Security Testing

## 1. JSON Web Token (JWT) Signature Architecture

A JSON Web Token consists of three Base64URL-encoded components separated by dots:
```
Header.Payload.Signature
```

The header specifies the cryptographic algorithm used to sign and verify the token:
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

Two common algorithm paradigms are:
1. **Asymmetric (RS256, ES256)**: The authentication server signs the token using its **Private Key**. Any resource server / microservice can verify the signature using the publicly shared **Public Key** (`.pem` / JWKS endpoint).
2. **Symmetric (HS256)**: Both signing and verification use the same **Shared Secret**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Algorithm Confusion Attack (CVE-2015-9235)           │
│                                                                        │
│   Target API: Expects RS256 with Public Key "PUBLIC_KEY.PEM"           │
│                                                                        │
│   Attacker Crafting:                                                   │
│   1. Extracts Public Key from https://auth.target.com/.well-known/jwks │
│   2. Changes Header: {"alg": "HS256"}                                  │
│   3. Signs Token using the PUBLIC KEY string as the HMAC SECRET!       │
│                                                                        │
│   Vulnerable Server Execution:                                         │
│   - Reads header: "alg = HS256"                                        │
│   - Executes: HMAC_SHA256(payload, PUBLIC_KEY_PEM)                     │
│   - Signature Matches! 💥 Full Admin Account Takeover!                 │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Common JWT Flaws Tested by QA Security Engineers

### A. The "None" Algorithm Bypass
- Early JWT libraries accepted `alg: "none"`, `alg: "None"`, or `alg: "NONE"`.
- If accepted, the server verifies the token without validating any signature!

### B. Algorithm Confusion (Asymmetric to Symmetric)
- If the verification library dynamically accepts the algorithm declared in the user's token header (`header.alg`) rather than strictly enforcing `algorithms: ['RS256']`, an attacker signs an administrative payload using the server's public key as an HMAC secret.

### C. `kid` (Key ID) Header Injections
- **Path Traversal (`kid: "../../dev/null"`)**: If the server loads the signing key from disk based on `kid`, directing it to an empty file (`/dev/null`) allows signing with an empty HMAC string `""`.
- **SQL Injection in `kid`**: Extracting keys from databases via unparameterized lookups.

---

## 3. Automated JWT Security Pen-Testing Script (Node.js)

Below is an automated test suite demonstrating how security QA engineers test backend verification against `alg: none` and RS256/HS256 confusion:

```javascript
const jwt = require('jsonwebtoken');
const axios = require('axios');
const fs = require('fs');
const { expect } = require('chai');

const TARGET_API = 'https://api.staging.example.com/v1/admin/users';

describe('JWT Cryptographic Verification & Alg Confusion Security Suite', () => {
  const publicKey = fs.readFileSync('./certs/public.pem', 'utf8');

  it('Vulnerability Test 1: Server must reject tokens with alg: none', async () => {
    // 1. Craft payload with forged admin privileges
    const header = Buffer.from(JSON.stringify({ alg: 'none', typ: 'JWT' })).toString('base64url');
    const payload = Buffer.from(
      JSON.stringify({
        sub: 'usr_attacker',
        role: 'SUPER_ADMIN',
        exp: Math.floor(Date.now() / 1000) + 3600,
      })
    ).toString('base64url');

    // Token with empty signature component
    const forgedToken = `${header}.${payload}.`;

    try {
      await axios.get(TARGET_API, {
        headers: { Authorization: `Bearer ${forgedToken}` },
      });
      expect.fail('Server accepted token with alg: none! Critical Security Vulnerability!');
    } catch (err) {
      expect(err.response.status).to.equal(401);
    }
  });

  it('Vulnerability Test 2: Server must reject RS256 -> HS256 key confusion attack', async () => {
    // 2. Sign token with HS256 using the server's public key as the secret
    const forgedPayload = {
      sub: 'usr_attacker',
      role: 'SUPER_ADMIN',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 3600,
    };

    const forgedToken = jwt.sign(forgedPayload, publicKey, {
      algorithm: 'HS256', // Attacker switches algorithm to symmetric HMAC
    });

    try {
      await axios.get(TARGET_API, {
        headers: { Authorization: `Bearer ${forgedToken}` },
      });
      expect.fail('Server vulnerable to Algorithm Confusion attack (CVE-2015-9235)!');
    } catch (err) {
      expect(err.response.status).to.equal(401);
    }
  });
});
```

---

## 4. Secure Implementation Standards for Developers

When logging security defects for JWT issues, recommend these secure backend patterns:

```javascript
// SECURE: Strictly enforce expected algorithm and explicitly bind public key
jwt.verify(token, publicKey, {
  algorithms: ['RS256'], // Hardcoded whitelist - never trust header.alg!
  issuer: 'https://auth.company.com',
  audience: 'https://api.company.com',
  clockTolerance: 10,
});
```

---

## 5. QA Verification Checklist

- [ ] **Algorithm Whitelist**: Confirm the server strictly specifies `algorithms: ['RS256']` during verification.
- [ ] **Reject Unsigned Tokens**: Verify that tokens lacking a signature or specifying `none` return HTTP `401 Unauthorized`.
- [ ] **Signature Tampering**: Modify a single byte of the payload or signature and ensure the token is rejected.
- [ ] **Expiration Enforcement**: Confirm expired tokens (`exp < currentTime`) return `TokenExpiredError` rather than being accepted.
