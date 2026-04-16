# Docker Build & Run Guide

This guide covers two scenarios:
- **Local testing** — build images from source on your dev machine
- **Live deployment** — update the running instances on the VM

---

## Local Testing

### Prerequisites — Temp PostgreSQL & Network

The live VM runs PostgreSQL directly on the host. For local testing, spin up a temporary container instead.

```bash
docker network create orcid-test-net

docker run --rm -d --name orcid-test-pg \
  --network orcid-test-net \
  -e POSTGRES_PASSWORD=test \
  postgres:15.2

# Confirm it's ready before continuing
docker run --rm --network orcid-test-net postgres:15.2 \
  pg_isready -h orcid-test-pg -U postgres

# Keys directory — run-app auto-generates self-signed certs here on first run
mkdir -p /tmp/orcid-test-keys
```

---

### 1. Production Image (`Dockerfile`)

```bash
# Build
docker build --target orcidhub -t orcidhub/app .
# or
make build

# Run
docker run --rm -p 8080:80 -p 8443:443 \
  --network orcid-test-net \
  -e DATABASE_URL="postgresql://postgres:test@orcid-test-pg:5432/postgres" \
  -v /tmp/orcid-test-keys:/.keys \
  -v $(pwd):/src \
  -v $(pwd):/var/www/orcidhub \
  orcidhub/app

# Test
curl -I http://localhost:8080      # expect 302 redirect to HTTPS
curl -sk https://localhost:8443    # expect NZ ORCID Hub HTML
```

---

### 2. Dev Image via docker-compose (`Dockerfile.dev`)

`docker-compose.yml` uses `image: orcidhub/app-dev:latest` — the same image family as the live
deployments. You must build it locally first with `make build-dev` before running Compose.

```bash
# Build (production base first, then dev layer — make build-dev does both)
make build-dev

# Run (full stack — includes containerised postgres and redis via standalone profile)
docker compose --profile standalone up -d
docker compose ps
curl -I http://localhost

# Tear down (must use same profile to stop db and redis)
docker compose --profile standalone down
```

---

### 3. Dev Image (`Dockerfile.dev`)

> The production image must be built first — `Dockerfile.dev` layers on top of it.

```bash
# Build (production first, then dev)
docker build --target orcidhub -t orcidhub/app .
docker build --target orcidhub-dev -f Dockerfile.dev -t orcidhub/app-dev .
# or
make build-dev  # automatically builds production first

# Run
docker run --rm -p 8080:80 -p 8443:443 \
  --network orcid-test-net \
  -e DATABASE_URL="postgresql://postgres:test@orcid-test-pg:5432/postgres" \
  -v /tmp/orcid-test-keys:/.keys \
  -v $(pwd):/src \
  -v $(pwd):/var/www/orcidhub \
  orcidhub/app-dev

# Test
curl -I http://localhost:8080      # expect 302 redirect to HTTPS
curl -sk https://localhost:8443    # expect NZ ORCID Hub HTML
```

---

### Cleanup

```bash
docker stop orcid-test-pg
docker network rm orcid-test-net
rm -rf /tmp/orcid-test-keys
```

---

## Live Deployment (VM)

The three live instances (orcidhub, test, dev) each have their own directory on the VM containing a `docker-compose.yml` and supporting files. They all use pre-built images pulled from Docker Hub — they do not build from source.

### Deploying a New Image Version

**Step 1 — Build, tag, and push from your dev machine:**

```bash
# Build all images
make build       # orcidhub/app
make build-dev   # orcidhub/app-dev (builds app first automatically)

# Tag with version
make tag         # orcidhub/app:8.0
make tag-dev     # orcidhub/app-dev:8.0

# Push to Docker Hub
make push        # pushes app:8.0, app-dev:8.0, app:latest, app-dev:latest
# or just dev image
make push-dev
```

**Step 2 — On the VM, for each instance directory:**

```bash
# Pull the new image
docker compose pull

# Restart with the new image
docker compose up -d
```

> Each instance directory (`orcidhub/`, `test/`, `dev/`) has its own `docker-compose.yml`
> with its own environment variables (different `ENV`, ports, domain names, ORCID credentials etc).
> These files stay on the VM and are not part of this repository.

---

## Key Differences: Local vs Live

| | Local Testing | Live VM |
|---|---|---|
| **Image source** | Built locally via `make build-dev`, referenced as `orcidhub/app-dev:latest` | Pulled from Docker Hub via `docker compose pull` (`orcidhub/app-dev:8.0`) |
| **PostgreSQL** | Temporary Docker container | Runs on the VM host, mounted via `/var/run/postgresql` |
| **Redis** | Docker container (via docker-compose) or none | Runs on the VM host, referenced by `RQ_REDIS_URL` |
| **SSL certs** | Auto-generated self-signed in `/tmp/orcid-test-keys` | Real certs in `.keys/` directory in each instance folder |
| **`app.conf`** | Minimal (HTTP only) from `conf/app.conf` in repo | Full production config mounted from `./app.conf` in instance folder |
| **Source code** | Mounted from repo (`-v $(pwd):/src`) | Mounted from instance directory on VM |
| **`DATABASE_URL`** | TCP connection with explicit port (required by `run-app` parser) | Socket path (`postgresql://orcidhub@/orcidhub?host=/run/postgresql`) or omitted to use socket auto-detection |
| **Ports** | `8080:80` (avoids conflicts) | `80:80`, `443:443` |
| **Restart policy** | None (`--rm`) | `restart: always` |

### Important Local Testing Gotchas

- **`DATABASE_URL` must include an explicit port** (e.g. `@host:5432/db`). The `run-app` script parses the host and port with a regex — a URL without a port will cause it to fall back to `db:5432` which won't resolve locally.
- **`/.keys` must be a mounted directory**, even if empty. `run-app` tries to write generated certs there and will error if the path doesn't exist. The generated certs are reused on subsequent runs.
- **`/src` must be mounted** so that `flask initdb` (run on every startup) can find the app code.
- The repo's `docker-compose.yml` is for local use only and is not used on the VM.
