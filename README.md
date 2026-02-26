# go-auth-service

A multi-tenant OAuth 2.0 authentication service in Go. Handles app installation flows and issues JWT credentials on behalf of a third-party identity provider. Designed for headless frontend architectures where the upstream platform doesn't natively issue portable tokens.

## What it does

1. **OAuth app installation** — two-leg OAuth 2.0 handshake to obtain platform API credentials, persisted per tenant.
2. **Credential bridging** — validates customer credentials against the upstream platform API, maps them to local identities, and issues HS256 JWT access + refresh token pairs.
3. **Multi-tenant routing** — serves multiple upstream stores from a single instance, selecting tenant credentials from the `Origin` request header.

## Architecture

```
cmd/main.go                           Entry point
internal/app/app.go                   Composition root — wires all dependencies
internal/controller/http/             HTTP layer (Gin): routes, middleware, error handling
internal/service/                     Business logic + interface definitions (ports)
internal/storage/                     GORM persistence (PostgreSQL)
internal/api/                         External vendor API adapters
internal/entity/                      Domain entities
pkg/errs/                             Typed errors with machine-readable codes
pkg/httpserver/                       HTTP server with graceful shutdown
pkg/logging/                          Logger interface + Zap implementation
pkg/postgresql/                       GORM connection setup
pkg/token/                            JWT signing and verification
```

The service layer owns all interfaces — `CustomerStorage`, `StoreStorage`, `VendorAPI`. Concrete implementations satisfy them from `storage/` and `api/`. The HTTP layer depends only on service interfaces, never on storage or vendor adapters directly.

## API

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/ping` | — | Health check |
| `POST` | `/customers/login` | — | Validate credentials against upstream; returns JWT |
| `POST` | `/customers/refresh-token` | `Bearer` | Issue new access token using stored refresh token |
| `GET` | `/customers/me` | `Bearer` | Return authenticated customer profile |
| `GET` | `/vendors/install` | — | Begin OAuth app installation |
| `GET` | `/vendors/redirect` | — | OAuth callback; exchanges code for API credentials |

## Token model

- **Access token**: HS256 JWT, 1h TTL, carries `userId` in payload.
- **Refresh token**: HS256 JWT, 24h TTL, stored server-side on the customer record (enables revocation).

## Tech stack

Go, Gin, GORM, PostgreSQL, JWT (golang-jwt/v5), Zap, Docker.

## Running

```sh
cp config/.env.example config/.env
# fill in your values
docker-compose up --build
```

The API is available at `http://localhost:8080`.

## Adding a vendor backend

Implement the `VendorAPI` interface defined in `internal/service/apis.go`, wire it into `internal/app/app.go`, and the rest of the service layer requires no changes.
