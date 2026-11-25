## My role (Security layer, internship at NetWaciila)
- Built a private PKI (RSA-4096 Root CA, per-service certs) and mTLS on all 5 services
- Implemented HMAC-SHA256 request signing + Redis nonce anti-replay (backend-side)
- JWT auth: global gateway filter + defense-in-depth validation on every backend
- RBAC: role matrix at gateway + [Authorize(Roles)] on backends, fail-closed
- Redis token-bucket rate limiting; fail-closed secret management
- Security audit: Trivy (CVEs fixed), OWASP ZAP (3 scans); load tests with JMeter
![Secure API Gateway architecture](docs/architecture_gateway.png)
## Not my scope
Gateway routing config and CI/CD: @Hafidamesad
