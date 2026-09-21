# buy-01 — E-Commerce Microservices Platform

An end-to-end e-commerce marketplace built with **Spring Boot microservices** on the backend and **Angular** on the frontend. Clients browse and search products, add them to a cart, and check out; sellers manage their own catalog, product images, and stock. The platform is fully containerized and ships with a **Jenkins CI/CD pipeline** and **SonarQube** static analysis integration.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Environment Variables](#environment-variables)
- [Running Locally](#running-locally)
- [API Overview](#api-overview)
- [Security](#security)
- [Testing](#testing)
- [CI/CD Pipeline (Jenkins)](#cicd-pipeline-jenkins)
- [Static Code Analysis (SonarQube)](#static-code-analysis-sonarqube)
- [Deployment & Rollback](#deployment--rollback)
- [Notifications](#notifications)
- [Scripts](#scripts)

---

## Architecture

The system is composed of independently deployable Spring Boot services fronted by a gateway, plus an Angular single-page application:

```
marketplace-ui (Angular SPA, Nginx, HTTPS)
      │
      ▼
   gateway (Spring Cloud Gateway, JWT validation + CORS, forwards X-User-Id/X-User-Role)
      │
      ├──▶ user service    ──▶ users-mongo
      ├──▶ product service ──▶ products-mongo  ──▶ (Kafka) ──▶ search index
      ├──▶ media service   ──▶ media-mongo  +  Cloudflare R2 (image files)
      ├──▶ orders service  ──▶ orders-mongo         (cart + checkout, calls product service to adjust stock)
      └──▶ search service  ──▶ Elasticsearch        (fed by Kafka events from product service)

   discovery         — Eureka service registry; every service above registers with it
   Kafka + Zookeeper — async event bus (currently: product lifecycle → search index)
```

- **discovery** — Eureka server for service registration and discovery.
- **gateway** — Single entry point (Spring Cloud Gateway, reactive/WebFlux); validates the JWT and injects `X-User-Id` / `X-User-Role` headers for downstream services, applies CORS.
- **user** — Authentication (`/api/auth/**`: signup/login), profiles and roles (`/api/users/**`: `CLIENT`, `SELLER`).
- **product** — Product CRUD, ownership enforcement, stock updates on checkout/cancel, publishes Kafka events consumed by the search service.
- **media** — Image upload/update/download via Cloudflare R2 object storage, MIME/size validation (≤ 2 MB, via Apache Tika content sniffing).
- **orders** — Shopping cart (`/api/cart/**`) and order placement/checkout (`/api/orders/**`); simulates order-status progression (confirmed → shipped → delivered) and exposes basic sales analytics.
- **search** — Product search backed by **Elasticsearch**; its index is kept in sync via Kafka events published by the product service.
- **marketplace-ui** — Angular SPA with route guards, HTTP interceptors, and reactive forms; feature modules for auth, products, cart, checkout, orders, seller and profile.
- **Kafka** — Backbone for asynchronous events (product lifecycle → search index) so services stay decoupled.
- Each service maintains its **own MongoDB database** (database-per-service, except `search`, which uses Elasticsearch), and exposes `/actuator/health` for observability.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java, Spring Boot, Spring Cloud Gateway (WebFlux), Spring Security, Spring Data MongoDB |
| Service Discovery | Netflix Eureka |
| Messaging | Apache Kafka + Zookeeper |
| Database | MongoDB (one instance per service: users, products, media, orders) |
| Search | Elasticsearch (product search index, populated via Kafka) |
| Object Storage | Cloudflare R2 (S3-compatible) for media files |
| Frontend | Angular (standalone components, signals, Reactive Forms) |
| Auth | JWT issued/validated by the Gateway, propagated to downstream services via `X-User-Id` / `X-User-Role` headers |
| Containerization | Docker, Docker Compose |
| CI/CD | Jenkins (declarative pipeline, distributed backend/frontend agents) |
| Code Quality | SonarQube |
| Web Server (frontend) | Nginx, self-signed TLS for local HTTPS |

---

## Project Structure

```
buy-01
├── Backend
│   ├── discovery/        # Eureka service registry
│   ├── gateway/           # API Gateway, JWT filter, routing config
│   ├── user/               # Auth, profiles, roles
│   ├── product/           # Product CRUD, ownership checks, stock updates
│   ├── media/               # Image upload/validation, R2 storage
│   ├── orders/              # Cart + order placement, checkout, analytics
│   ├── search/              # Product search (Elasticsearch, Kafka-fed)
│   └── jenkins/            # Jenkins master + backend/frontend agent images
├── marketplace-ui/         # Angular SPA
├── scripts/                 # Helper shell scripts (see Scripts section)
├── docker-compose.yml         # Application services (discovery, gateway, user, product, media, orders, search, marketplace-ui)
├── docker-compose.infra.yml   # Infra: MongoDB instances, Kafka/Zookeeper, Elasticsearch, Jenkins, SonarQube
├── docker-compose.local.yml   # Local dev: app services + MongoDB + Kafka bundled (no Jenkins/SonarQube/search)
└── Jenkinsfile              # CI/CD pipeline definition
```

---

## Prerequisites

- Docker & Docker Compose
- Java 21+ and Maven (for local backend development outside containers)
- Node.js + npm (for local Angular development)
- A `.env` file at the project root (see below)

---

## Environment Variables

Create a `.env` file at the project root (see `.env.example` for the full list). Key variables include:

```env
# MongoDB
DB_USERNAME=
DB_PASSWORD=
USERS_DB_NAME=
PRODUCTS_DB_NAME=
ORDERS_DB_NAME=
USER_DB_URI=
PRODUCT_DB_URI=
MEDIA_DB_URI=
ORDERS_DB_URI=

# JWT
JWT_SECRET=
JWT_EXPIRATION=

# CORS
CORS_ALLOWED_ORIGIN=https://localhost:4200

# SSL (Gateway HTTPS keystore, generated by scripts/create_Self-Signed-Certificate.sh)
SSL_KEYSTORE_PASSWORD=

# Cloudflare R2 (media storage)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
R2_BUCKET=
R2_ENDPOINT=
R2_PUBLIC_URL=

# Local dev
USER_ID=

# Jenkins / SonarQube (CI infra)
JENKINS_ADMIN_USERNAME=
JENKINS_ADMIN_PASSWORD=
GITHUB_USERNAME=
GITHUB_TOKEN=
SMTP_HOST=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
JENKINS_URL=
BACKEND_SLAVE_SECRET=
FRONTEND_SLAVE_SECRET=
NGROK_TOKEN=
```

> Never commit `.env` — it is git-ignored. Use `.env.example` as the template. Elasticsearch requires no credentials in this setup (`xpack.security.enabled=false`); its URL is hardcoded in `docker-compose.yml`, not read from `.env`.

---

## Running Locally

### 1. Create your `.env` file

```bash
cp .env.example .env
# then fill in DB credentials, JWT_SECRET, SSL_KEYSTORE_PASSWORD, R2 credentials, etc.
```

### 2. Generate local TLS certificates (Gateway HTTPS)

```bash
chmod +x scripts/create_Self-Signed-Certificate.sh
./scripts/create_Self-Signed-Certificate.sh
```

### 3. Start the stack

**Option A — local dev stack** (app services + MongoDB + Kafka bundled, simplest; does **not** include `search`/Elasticsearch):

```bash
docker compose -f docker-compose.local.yml up -d --build
```

**Option B — full stack** (mirrors CI/prod: app services + MongoDB + Kafka + Elasticsearch + Jenkins + SonarQube):

```bash
docker compose -f docker-compose.yml -f docker-compose.infra.yml --profile infra up -d --build
```

### 4. Access the app

| Service | URL |
|---|---|
| Frontend (Angular) | https://localhost:4443 |
| Gateway | https://localhost:8443 |
| Eureka Dashboard | http://localhost:8761 |

### Stopping / cleaning up

```bash
docker compose -f docker-compose.local.yml down
# or, for the full stack:
docker compose -f docker-compose.yml -f docker-compose.infra.yml down
```

> `scripts/clear.sh` also exists, but it is **not scoped to this project** — it stops/removes *all* containers, images, volumes and networks on the machine (`docker system prune -a --volumes -f`). Only run it on a machine where a full Docker reset is acceptable.

---

## API Overview

All external traffic goes through the **gateway** (`https://localhost:8443`), which validates the JWT and forwards `X-User-Id` / `X-User-Role` headers to the appropriate downstream service.

**User Service** (`/api/auth/**`, `/api/users/**`)
- `POST /api/auth/signup` — register as `CLIENT` or `SELLER`
- `POST /api/auth/login` — returns JWT
- `GET /api/users/me` / `PUT /api/users/me` — current user's profile
- `GET /api/users/all`, `GET /api/users/{id}`, `DELETE /api/users/{id}`

**Product Service** (`/api/product/**`)
- `GET /api/product`, `GET /api/product/{id}` — public
- `POST /api/product`, `PUT /api/product/{id}`, `DELETE /api/product/{id}` — seller-only, ownership enforced
- `GET /api/product/myProducts` — seller's own catalog
- `PATCH /api/product/update-stock` / `PATCH /api/product/restock-stock` — bulk stock adjustment on checkout/cancel (called internally by the Orders Service)

**Media Service** (`/api/media/**`)
- `POST /api/media/images` — seller-only, validates `image/*` MIME type (Apache Tika content sniffing) and 2 MB limit
- `PUT /api/media/images` — replace an existing image
- `GET /api/media/images/{id}` — serves image with caching headers
- `DELETE /api/media/images/{id}` — seller must own the media

**Orders & Cart Service** (`/api/orders/**`, `/api/cart/**`)
- `GET /api/cart`, `POST /api/cart/items`, `PATCH /api/cart/items`, `DELETE /api/cart/items/{productId}`, `DELETE /api/cart`
- `POST /api/orders` — place an order from the cart
- `GET /api/orders`, `GET /api/orders/{id}` — list / view own orders
- `PATCH /api/orders/{id}/cancel`, `DELETE /api/orders/{id}`
- `GET /api/orders/analytics?period=` — sales analytics
- Any authenticated user (client or seller) can call these endpoints; access is scoped by `X-User-Id`, not by an additional role check.

**Search Service** (`/api/search/**`)
- `GET /api/search/products?keyword=&category=&minPrice=&maxPrice=&sortBy=&page=&size=` — public, queries the Elasticsearch index kept in sync via Kafka events from the Product Service.

All services expose **`/actuator/health`** for liveness/readiness checks.

---

## Security

- **JWT** issued by the User Service, validated at the Gateway (`JwtFilter`), which injects `X-User-Id` / `X-User-Role` headers; downstream services trust these headers (`HeaderAuthFilter`) rather than re-parsing the JWT.
- **BCrypt** password hashing — passwords are never exposed in responses.
- **Ownership enforcement** — sellers can only modify/delete their own products and media (`sellerId == auth.subject`). Orders/cart endpoints are authenticated-only (any `CLIENT` or `SELLER`), scoped by user id — no additional role restriction.
- **File validation** — Media Service validates MIME type via content sniffing (Apache Tika) and enforces the 2 MB size limit, rejecting non-image payloads.
- **CORS** — enforced at the Gateway via `CORS_ALLOWED_ORIGIN`.
- **HTTPS** — Gateway and frontend both terminate TLS locally via a self-signed certificate/keystore (`keystore.p12`, `ssl/`); use Let's Encrypt in production.
- **Global exception handling** — each service has a `GlobalExceptionHandler` mapping errors to proper status codes (400/401/403/404) instead of leaking unhandled 5xx errors.

---

## Testing

- **Backend** — JUnit + Mockito per service (`ProductServiceTest`, `MediaServiceTest`, `UsersServiceTest`, controller tests with `@WebMvcTest`, etc.), run via `mvn clean package`.
- **Frontend** — Jasmine/Karma specs alongside each component/service (`*.spec.ts`), run via `npm test -- --watch=false --no-progress`.
- The Jenkins pipeline **fails the build** if any test fails, and publishes JUnit results via `junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'`.

---

## CI/CD Pipeline (Jenkins)

The `Jenkinsfile` implements a declarative pipeline with **distributed agents** (`backend` and `frontend` labels) and **change-based execution** — only services actually touched by a commit are built, tested, analyzed, and redeployed.

### Pipeline stages

1. **Checkout Source Code** — clones the repo on a `backend` agent, stashes the workspace so the `frontend` agent can reuse it without re-cloning.
2. **Detect Which Services Changed** — runs `scripts/detect-changed-services.sh` against the diff (PR target branch or `HEAD~1`), reading the real service list from `docker-compose.yml`, to compute `CHANGED_SERVICE_NAMES`.
3. **Build And Test** *(parallel)*:
   - **Backend Services** — for each changed backend service (`discovery`, `gateway`, `user`, `product`, `media`, `orders`, `search`): `mvn clean package` + JUnit report publishing.
   - **Frontend Application** — if `marketplace-ui` changed: `npm ci`, `npm test --coverage`, `npm run build -- --configuration production`.
4. **Static Code Analysis** *(parallel)* — SonarQube scan per changed backend service (`mvn sonar:sonar -Dsonar.projectKey=buy01-<service>`, e.g. `buy01-orders`, `buy01-search`) and `sonar-scanner` for the frontend, both wrapped in `withSonarQubeEnv('sonarqube-server')`.
5. **Quality Gate** — blocks the pipeline (5-minute timeout, `abortPipeline: true`) until SonarQube reports pass/fail.
6. **Build Docker Images** — `docker compose --profile infra -f docker-compose.yml -f docker-compose.infra.yml build <service>` per changed service, tagged with the short commit SHA (`IMAGE_TAG=<7-char-sha>`).
7. **Deploy To Main Environment** — on the `main` branch only: `docker compose --profile infra -f docker-compose.yml -f docker-compose.infra.yml up -d --no-deps discovery gateway product user media search orders marketplace-ui`. Note this redeploys **all** application services on every push to `main`, not just the ones changed.

### Triggers

- Build triggers are configured in the Jenkins job (e.g. GitHub webhook / poll SCM) to start automatically on new commits.
- Parameterized/matrix-style builds are achieved through the per-service `each { serviceName -> ... }` loops driven by change detection.

### Notifications

- The pipeline's `post { success / failure }` blocks send **email notifications** to the configured recipients, including the affected services and a link to the full build log/console output.

---

## Static Code Analysis (SonarQube)

SonarQube runs via Docker Compose (`docker-compose.infra.yml`, `sonarqube` + `sonarqube-db` services):

```bash
docker compose --profile infra -f docker-compose.infra.yml --env-file .env up -d sonarqube-db sonarqube
```

- Dashboard: `http://localhost:9001` (mapped from container port 9000).
- Each backend service is registered as its **own SonarQube project** (`buy01-discovery`, `buy01-gateway`, `buy01-user`, `buy01-product`, `buy01-media`, `buy01-orders`, `buy01-search`), and the frontend as `marketplace-ui`, so quality metrics are tracked independently per service.
- Authentication to SonarQube from Jenkins uses a stored credential (`sonarqube-token`).
- The **Quality Gate** stage in the Jenkinsfile aborts the pipeline if a service fails its gate (major vulnerabilities, code smells, coverage/duplication thresholds), preventing low-quality or insecure code from reaching the deploy stage.
- Recommended process: configure a GitHub webhook (or GitHub Actions) so PRs/branches trigger analysis automatically, and require the quality gate + a code review approval before merging.

---

## Deployment & Rollback

- Deployment is driven by the **Deploy To Main Environment** stage, restricted to the `main` branch; it redeploys all application services (`docker compose up -d --no-deps <all services>`) regardless of which ones actually changed.
- Each image is tagged by **commit SHA** (`IMAGE_TAG`), so a rollback is a matter of re-running deployment with a previous known-good `IMAGE_TAG` (or reverting the commit and letting the pipeline redeploy), since older tagged images remain available in the image registry/build cache.
- Health checks (`/actuator/health` on each service, Mongo `healthcheck` in Compose) gate service startup ordering (`depends_on: condition: service_healthy`), reducing the chance of promoting a broken deployment.

---

## Notifications

Build and deployment results are emailed automatically:

- ✅ **Success** — recipients get the list of affected services and a link to the build log.
- ❌ **Failure** — recipients get a link directly to the console output for debugging.

Recipients are configured via the `NOTIFICATION_EMAIL_RECIPIENT` environment variable in the `Jenkinsfile`.

---

## Scripts

| Script | Purpose |
|---|---|
| `scripts/create_Self-Signed-Certificate.sh` | Generates the Gateway's self-signed TLS keystore (`keystore.p12`) via `keytool`, password from `SSL_KEYSTORE_PASSWORD` |
| `scripts/detect-changed-services.sh` | Diffs two git refs and prints which app services (from `docker-compose.yml`) changed — used by the Jenkins pipeline for change-based builds |
| `scripts/get_id.sh` | Exports the current host UID as `USER_ID`, for Docker volume permission alignment in local dev |
| `scripts/clear.sh` | **Destructive, machine-wide** Docker reset — stops/removes *all* containers, images, volumes and networks, not just this project's. Use `docker compose down` instead unless you intend a full reset. |

---

## Evaluation Checklist

- ⚙️ **Functionality** — role-based flows (CLIENT/SELLER), product & media CRUD, browsing/search, cart & checkout, order tracking
- 🔐 **Security** — JWT, BCrypt, ownership checks, CORS, TLS
- 🧩 **Architecture** — clean service boundaries, Eureka discovery, Kafka events, Gateway, Elasticsearch-backed search
- 🚫 **Reliability** — global exception handling, health checks, no unhandled 5xx
- 🎨 **UX** — responsive Angular UI, guards/interceptors, inline validation
- 🧪 **CI/CD** — automated build → test → analyze → deploy, quality gates, rollback via tagged images, email notifications
