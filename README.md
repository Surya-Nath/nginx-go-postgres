# Nginx + Go + PostgreSQL

A three-tier HTTP service: Nginx terminates port 80, a statically linked Go binary serves the API, PostgreSQL holds the data. The API image is built from `scratch` (~12 MB). Database port 5432 is not published.

[Repository](https://github.com/Surya-Nath/nginx-go-postgres)

---

## Architecture

```
Client :80
   └── Nginx          proxy_pass http://backend:8000
          └── Go API  postgres://…@db:5432/example
                 └── PostgreSQL 16
```

| Service | Image | Host port |
| --- | --- | --- |
| proxy | `week6-proxy` — Nginx + baked `nginx.conf` | **80** |
| backend | `week6-backend` — single Go binary on `scratch` | closed |
| db | `postgres:16` | **5432 closed** |

`image: postgres` (unpinned) tracks 18 and breaks the classic `/var/lib/postgresql/data` volume. This repo pins **16**.

---

## Backend image

```
FROM golang:1.22-alpine AS builder
  CGO_ENABLED=0
  go mod download
  go build -o /backend

FROM scratch
  COPY --from=builder /backend /backend
```

The compile toolchain never ships. `CGO_ENABLED=0` keeps the binary runnable without glibc. A builder-as-runtime image of the same code was ~500 MB; the `scratch` image is ~12 MB.

---

## Proxy image

Official `nginx` plus a bind-mounted conf only works when that file exists on disk. Production compose uses a one-line image instead:

```
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

Upstream is the Compose service name `backend:8000`, not `localhost`.

---

## Local

```bash
docker compose up -d --build
curl -s localhost
```

Expected body:

```json
["Blog post #0","Blog post #1","Blog post #2","Blog post #3","Blog post #4"]
```

Password for Postgres is Compose secret `db/password.txt` (`POSTGRES_PASSWORD_FILE`). Do not run the database container as `user: postgres` unless the secret mount is world-readable — the official entrypoint cannot open `/run/secrets/db-password` otherwise.

---

## CI

Push to `main` builds and pushes:

- `413816840602.dkr.ecr.ap-south-1.amazonaws.com/week6-proxy:$SHA`
- `413816840602.dkr.ecr.ap-south-1.amazonaws.com/week6-backend:$SHA`

GitHub Actions and Jenkins share that contract. PostgreSQL is pulled from Docker Hub on the target, not built.

---

## Deploy

`compose.prod.yaml` references the ECR tags above. Terraform creates one Ubuntu 22.04 instance in the **default VPC** and a security group (SSH from the operator IP, HTTP 80 to the world). Ansible installs Docker, copies `compose.prod.yaml` + `db/password.txt`, logs into ECR, and runs `compose up`.

No custom VPC, NAT, or load balancer.

---

## Layout

```
backend/Dockerfile     multi-stage → scratch
proxy/Dockerfile        Nginx + conf
proxy/nginx.conf
compose.yaml            local
compose.prod.yaml       ECR images
.github/workflows/      build-push
Jenkinsfile
terraform/              instance + SG
ansible/                install + up
```

`.pem`, Terraform state, and live inventory IPs are gitignored.
