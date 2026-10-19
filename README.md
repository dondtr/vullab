# VulLab — Vulnerable Authentication API for Learning

VulLab is a **deliberately vulnerable** web-security training lab, built as an
Express/Node.js REST API. It reproduces common authentication, JWT, and XML
vulnerabilities in the style of PortSwigger Web Security Academy labs so you can
study them hands-on in a safe, local environment.

Every lab ships in **two versions**:

| Version | Mount point | Purpose |
| ------- | ----------- | ------- |
| **v1** | `/api/v1/*` | **Vulnerable** — the flaw is present so you can exploit it. |
| **v2** | `/api/v2/*` | **Secure** — the same feature with the vulnerability fixed, for side-by-side comparison. |

---

## Tech Stack

- **Runtime:** Node.js + Express 5
- **Database:** PostgreSQL via Sequelize ORM
- **Sessions / cache:** Redis (`express-session` + `connect-redis`)
- **Auth:** `jsonwebtoken` (RS256 + HS256), `bcrypt` / `bcryptjs`
- **XML:** `libxmljs` (for the XXE / SSRF labs)
- **Views:** Pug + Tailwind CSS
- **API docs:** Swagger UI (`swagger-jsdoc` + `swagger-ui-express`)

---

## Project Structure

```
vullab/
├── app.js                  # Express entry point, mounts /api/v1 and /api/v2, Swagger UI
├── docker-compose.yml      # Redis + PostgreSQL services
├── scripts/
│   ├── generate-key.js     # Generates the RSA key pair used for JWT signing
│   └── sync-database.js     # Syncs Sequelize models (default / --alter / --force)
└── src/
    ├── apps/               # v1.app.js (vulnerable) & v2.app.js (secure) sub-apps
    ├── config/             # database, redis, session, jwt, email, swagger config
    ├── controllers/        # v1/ and v2/ request handlers
    ├── services/           # v1/ and v2/ business logic (where most flaws live)
    ├── middlewares/        # auth (session + JWT), validation, host, error handling
    ├── models/             # Sequelize models (user, product, order, cart, …)
    ├── routes/             # v1/ and v2/ route definitions + Swagger annotations
    ├── seeders/            # reference data + dummy data + admin bootstrap
    └── utils/              # jwt, email, response, session, AppError helpers
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- Docker (for Redis and PostgreSQL), or your own local instances

### 1. Install dependencies

```bash
npm install
```

### 2. Start the datastores

```bash
docker compose up -d
```

This brings up:
- **Redis** on `localhost:6379`
- **PostgreSQL** on `localhost:5432` (database `vulnerable_auth_db`)

### 3. Create a `.env` file

The app reads configuration from a `.env` file in the project root (it is
git-ignored). Minimal example:

```env
PORT=3500
NODE_ENV=development

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=vulnerable_auth_db
DB_USER=postgres
DB_PASSWORD=postgres

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis123
REDIS_SESSION_PREFIX=vulab:

# Sessions
SESSION_SECRET=change-me

# JWT
JWT_SECRET=change-me
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
JWT_ISSUER=vullab-api
JWT_AUDIENCE=vullab-client
JWT_KID=vullab-rs256-key-1

# Admin bootstrap (optional)
ADMIN_EMAIL=admin@vullab.com
ADMIN_PASSWORD=PleaseChangeMe123!

# Email (optional — for password-reset labs)
EMAIL_USERNAME=
EMAIL_PASSWORD=
```

### 4. Generate the RSA signing keys

The JWT labs sign RS256 tokens with a key pair written to `src/config/keys/`:

```bash
node scripts/generate-key.js
```

### 5. Sync the database schema

```bash
npm run db:sync          # create tables (default)
npm run db:sync:safe     # ALTER tables to match models
npm run db:sync -- --force   # DROP and recreate all tables (destroys data)
```

A default admin account (`admin` / `ADMIN_PASSWORD`) is created by the seeder.

### 6. Run the server

```bash
npm run dev     # with nodemon (auto-reload)
# or
npm start
```

Then open:

- API base: `http://localhost:3500/api/v1` and `/api/v2`
- **Swagger API docs:** `http://localhost:3500/api-docs`

The Swagger docs describe each endpoint, the root cause of the vulnerability,
and the attack steps — they are the primary guide for working through the labs.

---

## Labs Overview

### Authentication (`/auth`)

| Lab | v1 (vulnerable) path |
| --- | --- |
| Username enumeration — different responses | `/api/v1/auth/enum/different-responses` |
| Username enumeration — subtle differences | `/api/v1/auth/enum/subtle-responses` |
| Username enumeration — timing attack | `/api/v1/auth/enum/timing-responses` |
| Username enumeration — account lock | `/api/v1/auth/enum/account-lock` |
| Broken IP-based brute-force protection | `/api/v1/auth/brute-force/broken-ip-block` |
| Multiple credentials per request | `/api/v1/auth/brute-force/multiple-credentials-per-request` |
| "Stay logged in" cookie (offline brute-force) | `/api/v1/auth/brute-force/stay-logged-in-cookie` |
| Password-change brute-force | `/api/v1/auth/brute-force/password-change` |
| Password reset — broken logic | `/api/v1/auth/password-reset-broken-logic` |
| Password reset poisoning (`X-Forwarded-Host`) | `/api/v1/auth/password-reset-poisoning` |
| 2FA simple bypass | `/api/v1/auth/2FA/simple-bypass` |
| 2FA broken logic | `/api/v1/auth/2FA/broken-logic` |

The matching **secure** implementations live under `/api/v2/auth/*`, plus a
`POST /api/v2/auth/register` endpoint.

### JWT (`/jwt`)

| Lab | v1 (vulnerable) path |
| --- | --- |
| Unverified signature | `/api/v1/jwt/unverified-signature/*` |
| Flawed signature verification (`alg: none`) | `/api/v1/jwt/flawed-signature-verification/*` |
| Weak signing key (brute-forceable) | `/api/v1/jwt/weak-signing-key/*` |
| JWK header injection | `/api/v1/jwt/jwk-header-injection/*` |
| JKU header injection | `/api/v1/jwt/jku-header-injection/*` |
| KID header path traversal | `/api/v1/jwt/kid-header-injection/*` |
| Algorithm confusion (RS256 → HS256) | `/api/v1/jwt/algorithm-confusion/*` |

Each has `/login` and `/profile` sub-routes. Secure versions are under
`/api/v2/jwt/*`. The JWKS endpoint is served at
`/api/v{1,2}/.well-known/jwks.json`.

### XML — XXE & SSRF (`/xxe`)

| Lab | path |
| --- | --- |
| XXE to retrieve files | `POST /api/v1/xxe/retrieve-file/check-stock` |
| XXE to perform SSRF | `POST /api/v1/xxe/perform-SSRF-attacks/check-stock` |

### Intentional insecure defaults

- **v1 sub-app** sets `trust proxy = true`, so it blindly trusts
  `X-Forwarded-For` / `X-Real-IP` (used to bypass IP-based protections). The
  **v2 sub-app** leaves this at its secure default.
- A hardcoded weak JWT key (`secret1`) powers the weak-signing-key lab.

---

## NPM Scripts

| Script | Description |
| ------ | ----------- |
| `npm start` | Start the server (`node src/app.js`). |
| `npm run dev` | Start with nodemon; runs a DB sync first (`predev`). |
| `npm run db:sync` / `db:sync:safe` / `db:sync:create` | Sync Sequelize models. |
| `npm run db:redis` / `db:redis:down` / `db:redis:logs` | Manage the Redis container. |
| `npm run build:css` | Build Tailwind CSS. |
| `npm run lint` / `lint:fix` | Run ESLint. |
| `npm run format` / `format:check` | Run Prettier. |

---

## License

ISC. Author: Don. See `NAMING_GUIDE.md` for the project's naming conventions.
