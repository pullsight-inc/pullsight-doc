# Pullsight

Pullsight reviews pull requests with an LLM that sees the repository context, your review rules, and the diff. The official website is [pullsight.dev](https://pullsight.dev). A self-host install runs the database, the API, the review worker, and the web dashboard on your own host.

The supported pin is an immutable release tag, `vX.Y.Z` or a prerelease such as `vX.Y.Z-rc.N`. Use the same tag for all three images. `latest` moves only when a stable release is published; do not pin production to it.

Images:

| Image | Role |
|---|---|
| `ghcr.io/pullsight-inc/pullsight-db` | PostgreSQL with pgvector |
| `ghcr.io/pullsight-inc/pullsight-api` | API and review worker |
| `ghcr.io/pullsight-inc/pullsight-web` | Web dashboard |

GHCR is the registry to install from. A release may also mirror the same tag and digest to `docker.io/pullsight-inc`. The images are MIT-licensed.

## Requirements

- Docker Engine 24+ and Docker Compose v2
- A network path to GHCR
- Disk for the images and the Postgres volume
- A public HTTPS URL for GitHub App webhooks and OAuth callbacks

`redis:7-alpine` is pulled from Docker Hub during install. It is not a Pullsight image.

## Install

Create a directory with the three files below, then pull and start. Replace `vX.Y.Z` with the release tag you intend to run.

`docker-compose.yml`:

```yaml
services:
  db:
    image: "${PULLSIGHT_IMAGE_PREFIX:-ghcr.io/pullsight-inc}/pullsight-db:${PULLSIGHT_IMAGE_TAG:?Set PULLSIGHT_IMAGE_TAG}${PULLSIGHT_DB_DIGEST:+@${PULLSIGHT_DB_DIGEST:-}}"
    pull_policy: missing
    container_name: pullsight-db
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-pullsight_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: ${POSTGRES_DB:-pullsight_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    pull_policy: missing
    container_name: pullsight-redis
    environment:
      REDIS_PASSWORD: ${REDIS_PASSWORD:?Set REDIS_PASSWORD in .env}
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD:?Set REDIS_PASSWORD in .env}"]
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "REDISCLI_AUTH=\"$$REDIS_PASSWORD\" redis-cli ping | grep PONG"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    image: "${PULLSIGHT_IMAGE_PREFIX:-ghcr.io/pullsight-inc}/pullsight-api:${PULLSIGHT_IMAGE_TAG:?Set PULLSIGHT_IMAGE_TAG}${PULLSIGHT_API_DIGEST:+@${PULLSIGHT_API_DIGEST:-}}"
    pull_policy: missing
    container_name: pullsight-api
    ports:
      - "${API_PORT:-8000}:8000"
    env_file:
      - .env
    environment:
      PULLSIGHT_DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER:-pullsight_user}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB:-pullsight_db}
      PULLSIGHT_OLLAMA_BASE_URL: ${PULLSIGHT_OLLAMA_BASE_URL:-http://host.docker.internal:11434}
      PULLSIGHT_API_HOST: 0.0.0.0
      PULLSIGHT_API_PORT: 8000
      PULLSIGHT_REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  worker:
    image: "${PULLSIGHT_IMAGE_PREFIX:-ghcr.io/pullsight-inc}/pullsight-api:${PULLSIGHT_IMAGE_TAG:?Set PULLSIGHT_IMAGE_TAG}${PULLSIGHT_API_DIGEST:+@${PULLSIGHT_API_DIGEST:-}}"
    pull_policy: missing
    command: ["arq", "pullsight.workers.review_worker.WorkerSettings"]
    env_file:
      - .env
    environment:
      PULLSIGHT_DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER:-pullsight_user}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB:-pullsight_db}
      PULLSIGHT_OLLAMA_BASE_URL: ${PULLSIGHT_OLLAMA_BASE_URL:-http://host.docker.internal:11434}
      PULLSIGHT_REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      PULLSIGHT_REVIEW_WORKER_CONCURRENCY: ${PULLSIGHT_REVIEW_WORKER_CONCURRENCY:-4}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  web:
    image: "${PULLSIGHT_IMAGE_PREFIX:-ghcr.io/pullsight-inc}/pullsight-web:${PULLSIGHT_IMAGE_TAG:?Set PULLSIGHT_IMAGE_TAG}${PULLSIGHT_WEB_DIGEST:+@${PULLSIGHT_WEB_DIGEST:-}}"
    pull_policy: missing
    container_name: pullsight-web
    ports:
      - "${WEB_PORT:-3000}:3000"
    env_file:
      - web.env
    environment:
      AUTH_URL: ${AUTH_URL:-http://localhost:3000}
      NEXTAUTH_URL: ${NEXTAUTH_URL:-http://localhost:3000}
      PULLSIGHT_API_URL: http://api:8000
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-http://localhost:8000}
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
```

`.env`:

```bash
PULLSIGHT_IMAGE_TAG=vX.Y.Z
PULLSIGHT_IMAGE_PREFIX=ghcr.io/pullsight-inc

# Optional. Same tag and digest on the Docker Hub mirror:
# PULLSIGHT_IMAGE_PREFIX=docker.io/pullsight-inc

# Optional digest pins from `docker buildx imagetools inspect`.
# Leave unset unless you want Compose to require that exact digest.
# PULLSIGHT_DB_DIGEST=sha256:...
# PULLSIGHT_API_DIGEST=sha256:...
# PULLSIGHT_WEB_DIGEST=sha256:...

POSTGRES_USER=pullsight_user
POSTGRES_PASSWORD=
POSTGRES_DB=pullsight_db
REDIS_PASSWORD=

PULLSIGHT_ENV=production
PULLSIGHT_DEBUG=false
PULLSIGHT_ALLOWED_ORIGINS=["http://localhost:3000"]
PULLSIGHT_APP_URL=http://localhost:3000

# openssl rand -base64 32
PULLSIGHT_SECRET_KEY=
# Same value in web.env. openssl rand -base64 32
PULLSIGHT_PROVISION_SECRET=
# python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
PULLSIGHT_ENCRYPTION_KEY=
# openssl rand -base64 16
PULLSIGHT_API_KEY_SALT=

# ollama, or a hosted provider that uses your own API key: openai | anthropic | gemini | azure_openai
# A hosted provider does not use Ollama.
PULLSIGHT_DEFAULT_LLM_PROVIDER=ollama
PULLSIGHT_OLLAMA_BASE_URL=http://host.docker.internal:11434
PULLSIGHT_OLLAMA_MODEL=gemma4:e2b

PULLSIGHT_OPENAI_API_KEY=
PULLSIGHT_OPENAI_MODEL=gpt-5.6-luna
PULLSIGHT_ANTHROPIC_API_KEY=
PULLSIGHT_ANTHROPIC_MODEL=
PULLSIGHT_GEMINI_API_KEY=
PULLSIGHT_GEMINI_MODEL=
PULLSIGHT_AZURE_OPENAI_API_KEY=
PULLSIGHT_AZURE_OPENAI_ENDPOINT=
PULLSIGHT_AZURE_OPENAI_DEPLOYMENT=

PULLSIGHT_GITHUB_APP_ID=
PULLSIGHT_GITHUB_PRIVATE_KEY=
PULLSIGHT_GITHUB_WEBHOOK_SECRET=

PULLSIGHT_SMTP_HOST=
PULLSIGHT_SMTP_PORT=587
PULLSIGHT_SMTP_USERNAME=
PULLSIGHT_SMTP_PASSWORD=
PULLSIGHT_SMTP_FROM_EMAIL=noreply@example.com
PULLSIGHT_SMTP_USE_TLS=true
PULLSIGHT_SMTP_USE_SSL=false
```

`web.env`:

```bash
# openssl rand -base64 32
AUTH_SECRET=
PULLSIGHT_API_URL=http://api:8000
# Same value as PULLSIGHT_PROVISION_SECRET in .env
PULLSIGHT_PROVISION_SECRET=

AUTH_URL=http://localhost:3000
NEXTAUTH_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:8000

AUTH_GITHUB_ID=
AUTH_GITHUB_SECRET=
```

Generate `POSTGRES_PASSWORD` and `REDIS_PASSWORD` before the first start. Compose refuses to start while either value is empty. Avoid `@`, `:`, `/`, `?`, and `#` in those passwords. Postgres applies `POSTGRES_PASSWORD` only when the data volume is first created.

Postgres and Redis stay on the Compose network. They are not published on the host.

```bash
docker compose pull
docker compose up -d
```

Open `http://localhost:3000`, sign in, and connect a repository.

To run more than one review worker:

```bash
docker compose up -d --scale worker=3
```

`docker compose pull` is required on each upgrade. `pull_policy: missing` makes a later `up` reuse the image already on the host.

## Choose a model

Set `PULLSIGHT_DEFAULT_LLM_PROVIDER` in `.env` to one provider. `ollama` runs a local model. Any other value is a bring-your-own-key option: Pullsight calls that provider with your API key, and Ollama is not installed.

### Ollama

Leave the provider as `ollama`. Install [Ollama](https://ollama.com/download) on the Docker host, then:

```bash
ollama pull gemma4:e2b
```

The sample `.env` points the API at `http://host.docker.internal:11434`. If Ollama runs on another machine, start it with `OLLAMA_HOST=0.0.0.0` and set `PULLSIGHT_OLLAMA_BASE_URL=http://<ollama-host>:11434`.

### Your own API key

Set `PULLSIGHT_DEFAULT_LLM_PROVIDER` to `openai`, `anthropic`, `gemini`, or `azure_openai`, and fill in that provider's key in `.env`. Leave the Ollama settings as they are. There is no Ollama install and no `ollama pull` for this choice.

## GitHub

Webhook URL:

`https://<your-public-api-host>/api/webhooks/github`

Set `PULLSIGHT_GITHUB_APP_ID`, `PULLSIGHT_GITHUB_PRIVATE_KEY`, and `PULLSIGHT_GITHUB_WEBHOOK_SECRET` in `.env`.

OAuth callback:

`https://<your-public-web-host>/api/auth/callback/github`

In the GitHub App settings under **Identifying and authorizing users**, add that redirect URI. Keep **Redirect on update** disabled unless you set a Setup URL and want GitHub to send users there after they update the app. Set `AUTH_URL` and `NEXTAUTH_URL` in `web.env` to the public web origin, and set the matching `AUTH_GITHUB_ID` and `AUTH_GITHUB_SECRET`.

Azure DevOps service hooks use `https://<your-public-api-host>/api/webhooks/azure-devops`. Azure DevOps does not sign those requests. Keep that URL on a private network or behind a restricted proxy. Sign-in codes for Azure DevOps need the SMTP settings in `.env`.

## Upgrade

Set `PULLSIGHT_IMAGE_TAG` to the new release, then:

```bash
docker compose pull
docker compose up -d
```

The `postgres_data` volume is kept. Use the same tag for the database, API, and web images.

## Verify a release

The Cosign certificate is the release workflow on `main`. Repeat for `pullsight-db` and `pullsight-web`. The digest printed by Cosign is the digest to pin.

```bash
cosign verify \
  --certificate-identity "https://github.com/pullsight-inc/pullsight/.github/workflows/release.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/pullsight-inc/pullsight-api:vX.Y.Z
```

## License

MIT License.
