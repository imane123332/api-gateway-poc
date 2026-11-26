# Secure API Gateway for Microservices

Security layer for a Spring Cloud Gateway protecting 5 .NET microservices.
Built during my internship at NetWaciila (2026).

![Secure API Gateway architecture](docs/architecture_gateway.png)

# Secure API Gateway for Microservices

Security layer for a Spring Cloud Gateway protecting 5 .NET microservices
(4 REST + 1 SOAP) of a university ERP simulation.
Built during my internship at NetWaciila (Agadir, 2026).



## My contribution (security layer)
- Private PKI (RSA-4096 Root CA, per-service certificates with SAN) and mTLS between the gateway and each of the 5 backends (client → gateway is server-side TLS 1.3)
- JWT authentication (global gateway filter + independent validation on each backend, algorithm pinned to HS256)
- Fail-closed RBAC (role matrix at the gateway + `[Authorize(Roles)]` on each backend)
- HMAC-SHA256 request signing + Redis nonce anti-replay (±5 min window)
- Redis token-bucket rate limiting; no default secrets (the app refuses to start without them)

## Team scope (not mine)
Gateway routing config and CI/CD: @Hafidamesad

## Security controls
| Threat (STRIDE) | Control | Implementation | Test |
|---|---|---|---|
| Spoofing | mTLS (gateway ↔ backends) + JWT | `GatewaySslConfig.java`, Kestrel `RequireCertificate` + `CustomRootTrust`, `JwtAuthenticationFilter.java`, `JwtBearer` on each backend | no / forged client cert → TLS alert 46, handshake refused (5/5 backends); missing JWT → 401 |
| Tampering + replay | HMAC-SHA256 over `METHOD\nPATH\nTIMESTAMP\nNONCE\nSHA256(BODY)` + Redis nonce (`SETNX`, TTL 5 min) | `HmacSigningFilter` (gateway), `HmacVerificationMiddleware` (each backend) | missing / forged signature → 401; replayed nonce → 401 (5/5 backends) |
| Repudiation | User identity in JWT claims (`sub`, `role`) + gateway request logging (route, status, latency) | `RoutingLogFilter`, `X-Gateway-Latency-Ms` | log lines per routed request. **Partial: no centralized, tamper-proof audit log yet** |
| Info disclosure | TLS 1.3, fail-closed secrets, env-var configuration, non-root containers | `${JWT_SECRET}` / `${HMAC_SECRET}` without default, startup exception if missing, `load-env.sh` | app refuses to start without secrets; Trivy and ZAP (see below) |
| DoS | Rate limiting, Redis token bucket (10 req/s, burst 20), keyed on `X-Client-Id` with IP fallback | `RequestRateLimiter` + `KeyResolver` | demo thresholds (burst 3, 1 req/s): rapid requests → 429, bucket refills after a 3 s pause |
| Elevation of privilege | Fail-closed RBAC | `RoleMatrixProperties`, `RoleAuthorizationFilter` | wrong role → 403 (`X-Auth-Reject-Reason`), unmapped route → 403 |

## Test results
| Tool | Scope | Result |
|---|---|---|
| Trivy | Maven (gateway) + NuGet dependencies, Docker images | `pom.xml`: 1 CRITICAL (`netty-handler`) → 0 after upgrading Netty to 4.2.16.Final; EF Core upgraded to 10.0.11; images run as non-root `appuser` |
| OWASP ZAP 2.17.0 | 3 passes: unauthenticated baseline, authenticated (ADMIN, auto-refreshed JWT), manual RBAC boundary checks | authenticated scan of `https://localhost:8443`: 0 High, 0 Medium, 2 Low, 1 Informational |
| JMeter | 4-scenario plan (single route, multi-route, cold start, percentiles); preliminary run: 642 samples, 0 % errors | avg 398 ms, median 343 ms, p90 456 ms, p95 633 ms, p99 1,581 ms, max 4,981 ms (single outlier); cold start ≈ 701 ms. SLA 1.3 s met on mean / median / p95, **not on p99 or the outlier** |
| Replay / RBAC protocol | per service | RBAC: correct role → 200, wrong role → 403, unmapped route → 403, admin override → 200. Replay: 401 on all 5 backends |

## How to run (development)
Prerequisites: JDK 17, .NET 10 SDK, Docker (for Redis), OpenSSL.

```bash
# Redis (rate limiting + nonces)
docker run -d --name redis-gateway -p 6379:6379 redis

# Secrets (never committed)
export JWT_SECRET="..."
export HMAC_SECRET="..."
export SPRING_PROFILES_ACTIVE=secure

# Gateway
cd gateway
mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=secure

# Backends (one terminal each)
cd Finance.Api && dotnet run     # same for Etudiants, Transport, Academique, RhPaie
```

```bash
# Get a token (demo users are defined in AuthController, simulation only)
TOKEN=$(curl -sk -X POST https://localhost:8443/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | sed 's/.*"token":"\([^"]*\)".*/\1/')

# Call a backend through the gateway
curl -k https://localhost:8443/api/finance/paiements/1 -H "Authorization: Bearer $TOKEN"

# Check that a backend rejects a client without certificate (expect a TLS alert)
printf 'GET /health HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | \
  openssl s_client -connect localhost:5101 -CAfile security/pki/root-ca/ca.crt -quiet
```

PKI material is generated locally with OpenSSL under `security/pki/`; private keys are git-ignored and never committed.


## Report
[Technical report (PDF)](docs/report.pdf.pdf)
