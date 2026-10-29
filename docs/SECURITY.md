# Security

A summary of how Sports-Better protects its users and its server. It has been through several audit waves, including a full multi-agent code and security review, with fixes tracked to completion.

Implementation details that would mostly help an attacker are left out on purpose.

## Accounts and sessions

- **Passwords** are hashed with Argon2id. Hashing runs on worker threads behind a semaphore, so a burst of logins can't block the event loop or exhaust the CPU. Older hashes are upgraded automatically on the next login.
- **Email verification** is required before an account can use the app, and there is one account per email address (case-insensitive), which blocks sign-up-bonus farming.
- **Verification and password-reset tokens** are single-use and expire.
- **Sessions** are signed, HTTP-only cookies with an expiry, backed by a server-side revocation list.
- **Account deletion and data export** are self-service. Deletion revokes the session first, cancels any active subscription, then purges the user's records.
- **Third-party API keys** that users connect are encrypted at rest.

## API and WebSockets

- Every API route requires a valid session except a short public allowlist (health, config, sign-in and sign-up, the billing webhook, and public pages such as the track record and legal documents).
- **WebSockets** authenticate the session and check the `Origin` header against an allowlist, which defends against cross-site WebSocket hijacking. Each user has a cap on concurrent sockets and concurrent analyses.
- **Rate limiting** applies per client IP on sensitive routes. The IP is taken from a header only the trusted edge can set.
- **Request bodies** have a size limit.
- **Security headers** are set on every response: `nosniff`, `X-Frame-Options: DENY`, `no-referrer`, and HSTS over HTTPS.
- **Authorization lives in the backend.** The UI hides admin pages, but the server enforces access on every route. Higher-risk owner tools have their own allowlist on top of the owner check.

## Payments

- Stripe webhooks are signature-verified and deduplicated by event ID.
- Billing state changes only through verified webhook events, never from client requests.
- No card data ever touches the server; checkout and the billing portal are hosted by Stripe.

## Data handling

- The game store and other frequently written JSON stores are written atomically (temp file, then rename), so a crash mid-write can't corrupt them.
- The usage database and credential files are created with owner-only file permissions.
- Secrets live in environment files outside version control.

## Infrastructure

- The backend container runs as an unprivileged user (dropped from root via `gosu`), with `no-new-privileges`, a memory limit and a process limit.
- The server has **no open inbound ports**. All traffic arrives through an outbound Cloudflare Tunnel, with Cloudflare's TLS and WAF in front.
- The web app is served from Cloudflare's edge, and its API proxy has a hardcoded upstream that requests can't redirect.
- Backend dependencies are pinned in a lock file, so every build is reproducible.
- CI runs linting (ruff, ESLint), type-checking (mypy on the security-sensitive modules), unit tests and a production build on every pull request.
