# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| latest  | Yes       |

## Reporting a Vulnerability

**Do NOT open a public GitHub issue for security vulnerabilities.**

Email **security@zerodaysec.com** with:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if known)

We will acknowledge within **48 hours** and aim to release fixes for critical issues within **30 days**.

## Credential Overview

| Variable | Sensitivity | Description |
|----------|-------------|-------------|
| `INSTAGRAM_ACCESS_TOKEN` | High | Long-lived Instagram Graph API access token |
| `FACEBOOK_APP_ID` | Medium | Public-facing Facebook App identifier |
| `FACEBOOK_APP_SECRET` | **Critical** | Used to sign API requests; compromise enables full app impersonation |
| `INSTAGRAM_BUSINESS_ACCOUNT_ID` | Low | Public Instagram account ID |

**Never commit any of these values to source control.** Use environment variables or a secrets manager.

---

## Facebook App Secret Hardening

`FACEBOOK_APP_SECRET` is the most sensitive credential in this integration. A leaked App Secret allows an attacker to:
- Generate valid access tokens for any user who has authorized your app
- Impersonate your application in all API calls
- Invalidate all existing user sessions by rotating the secret

### Required Hardening Measures

1. **Enable App Secret Proof (`appsecret_proof`)**

   All server-side API calls must include `appsecret_proof`, an HMAC-SHA256 hash of the access token signed with your App Secret:

   ```python
   import hashlib, hmac

   def get_appsecret_proof(access_token: str, app_secret: str) -> str:
       return hmac.new(
           app_secret.encode("utf-8"),
           access_token.encode("utf-8"),
           hashlib.sha256
       ).hexdigest()
   ```

   Append `&appsecret_proof=<hash>` to all Graph API requests. This prevents stolen access tokens from being used outside your server.

   Enable enforcement in Meta App Dashboard: **App Settings → Advanced → Require App Secret**.

2. **Restrict Allowed Domains**

   In **Meta App Dashboard → Facebook Login → Settings → Valid OAuth Redirect URIs**, restrict to only the domains your deployment actually uses. Do not leave wildcards or `localhost` in production settings.

3. **Store the App Secret in a Secrets Manager**

   Never pass `FACEBOOK_APP_SECRET` via `.env` files in production. Use AWS Secrets Manager, HashiCorp Vault, or equivalent. If using Docker/Kubernetes, inject via runtime secrets — not build-time environment variables.

4. **Rotate Immediately on Exposure**

   If your App Secret is committed to git, logged, or otherwise exposed:
   - Rotate immediately in Meta App Dashboard → **App Settings → Advanced → Reset App Secret**.
   - Invalidate all existing access tokens (all users will need to re-authorize).
   - Review Meta's audit logs for unauthorized API usage.

---

## Instagram Access Token Security

Access tokens expire after approximately 60 days. There is no automated refresh built into this server.

- **Proactively refresh tokens** before expiry using the long-lived token refresh endpoint:
  ```
  GET https://graph.facebook.com/refresh_access_token
    ?grant_type=fb_exchange_token
    &client_id={app_id}
    &client_secret={app_secret}
    &fb_exchange_token={current_token}
  ```
- Silent token expiry will cause all API calls to fail with `OAuthException` — monitor for this error in production.
- Store refreshed tokens back to your secrets store, not to disk or source control.

---

## Meta Advanced Access (Instagram DM Features)

Certain Instagram DM and messaging features require **Meta Advanced Access**, which must be approved by Meta before use:

- **Standard Access** is granted automatically and covers basic account, media, and insights operations.
- **Advanced Access** is required for `instagram_manage_messages`, `pages_messaging`, and other DM/inbox features.

**Warning:** Using Instagram DM tools in this server without approved Advanced Access violates Meta's Platform Terms of Service and may result in app suspension. Submit a review request via Meta App Dashboard → **App Review → Permissions and Features** before enabling DM functionality in production.

---

## Security Considerations

- This server passes Instagram business account data (posts, media, insights, DMs) to the AI assistant that invokes it. Ensure the AI assistant environment is operated in a trusted, access-controlled context.
- Instagram business data may contain proprietary analytics, customer DMs, and unpublished content. Apply appropriate data handling and retention controls in your deployment.
- Review and restrict API scopes in Meta App Dashboard to only the permissions this server actually uses.
