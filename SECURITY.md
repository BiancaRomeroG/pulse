# Security

## Scope
This repository is a portfolio project. It handles **no real money, no real personal data and no production credentials**. Providers, telemetry and tenants are simulated or seeded.

## Reporting
If you find a vulnerability, open a private security advisory on GitHub or email biancaromeroguz@gmail.com. Expect an answer within 7 days.

## Practices in this repo
- Secrets only via environment variables; `.env` is git-ignored; `.env.example` documents every variable.
- Inbound webhooks / events are verified with HMAC-SHA256, timestamp window and nonce replay protection.
- API keys are stored hashed (argon2). Plaintext is shown once.
- Dependencies audited in CI (`pnpm audit --prod`), Dependabot enabled.
- Docker images run as non-root.
- Security headers set in the web app (CSP, HSTS, X-Content-Type-Options, Referrer-Policy).
