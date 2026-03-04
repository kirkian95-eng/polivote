# Security Rules

- Never store plaintext passwords — use bcrypt or argon2 for hashing
- All API endpoints must validate authentication before processing
- Use parameterized queries exclusively — never interpolate user input into SQL
- Sanitize all user-facing output to prevent XSS
- Set secure, httpOnly, sameSite flags on all cookies
- Apply rate limiting to authentication and public-facing endpoints
- Log security-relevant events (login attempts, permission changes, vote submissions) with timestamps
- Never log sensitive data (passwords, tokens, PII beyond what's needed for audit)
- Use HTTPS everywhere — reject plain HTTP in production
- Keep dependencies updated; audit with `npm audit` regularly
