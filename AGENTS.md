# AGENTS.md

## Cursor Cloud specific instructions

### Project Overview

Sea-RideTheWind (识海) is a Go microservice backend built with the [go-zero](https://github.com/zeromicro/go-zero) framework. It consists of 12 services under `service/` (user, admin, article, comment, like, follow, favorite, message, task, hot, points, security), each with API (HTTP) and/or RPC (gRPC) sub-services.

### Key Gotchas

- **`go.mod` is gitignored.** You must run `go mod init sea-try-go && go mod tidy` before building. The module name is `sea-try-go`.
- **`volumes/` directory is Docker-created.** Running `go list ./...` or `go build ./...` from the repo root will fail with a permission error on `volumes/etcd/member`. Use `go build ./service/...` or `go vet ./service/...` instead.
- **Services must run from their own directory.** Log config paths are relative (e.g. `../../../../log/user`), so each service binary must be started from its own source directory (e.g. `cd service/user/user/rpc && ./binary -f etc/user.yaml`).
- **RPC services must start before API services.** API services discover RPC via etcd; start the RPC service first so it registers itself.

### Infrastructure

Start required infrastructure with Docker Compose (from repo root):

```
docker compose up -d etcd postgres redis kafka minio beanstalkd1 beanstalkd2
```

Key ports (mapped to localhost):
| Service | Port |
|---------|------|
| etcd | 32379 |
| PostgreSQL | 35432 (user: admin, pass: Sea-TryGo, db: first_db) |
| Redis | 36379 |
| Kafka | 39092 |
| MinIO | 39000 (API), 39001 (Console) |
| Beanstalkd | 41300, 41301 |

Service YAML configs already reference `127.0.0.1` with these ports.

### Build & Run

```bash
# Build all services (from repo root)
go build ./service/...

# Build a specific service
go build -o /tmp/user-api ./service/user/user/api/
go build -o /tmp/user-rpc ./service/user/user/rpc/

# Run (from the service's own directory)
cd service/user/user/rpc && /tmp/user-rpc -f etc/user.yaml
cd service/user/user/api && /tmp/user-api -f etc/usercenter.yaml
```

### Lint & Test

```bash
go vet ./service/...
go test ./service/common/...
go test ./service/hot/heavykeeper/...
```

### Service Build Paths (from manage.sh)

See `manage.sh` `resolve_build_paths()` for the mapping of service name to API/RPC build paths. For example, user API is `./service/user/user/api` and user RPC is `./service/user/user/rpc`.

### Docker-based Deployment

The `manage.sh` script handles Docker-based build and deployment. It requires `INFRA_HOST` to be set for the app role. For local development, running services natively (as above) is simpler.
