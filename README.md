
# VeritasID — Backend

> **The verification and issuance engine behind on-chain OSS contributor credentials.**

---

## Table of Contents

- [Project Overview](#project-overview)
- [Aim](#aim)
- [Objectives](#objectives)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [Available Scripts](#available-scripts)
- [Contributing](#contributing)
- [License](#license)

---

## Project Overview

The VeritasID Backend is a Node.js API service that acts as the trusted issuance authority and data orchestration layer for the VeritasID protocol. It is responsible for verifying the authenticity of GitHub contributions, constructing W3C-compliant Verifiable Credential payloads, invoking the VeritasID Soroban smart contracts to mint on-chain attestations, and serving credential and reputation data to the frontend and third-party integrators.

The backend enforces the protocol's trust boundary — no credential reaches the blockchain without passing through its verification pipeline, which validates contribution data against the live GitHub API before any issuance is authorised.

---

## Aim

To build a secure, auditable, and extensible backend service that forms the authoritative issuance and verification engine of the VeritasID protocol — ensuring every on-chain credential is backed by independently verifiable GitHub contribution data, and that the protocol remains trustworthy, transparent, and resistant to fraudulent credential claims.

---

## Objectives

- Implement a secure GitHub OAuth 2.0 authentication flow that links a developer's GitHub identity to their Stellar wallet address
- Build a contribution verification pipeline that queries the GitHub API to confirm the authenticity of a contribution before credential issuance is triggered
- Construct W3C Verifiable Credential payloads conforming to the VC Data Model 2.0 specification prior to on-chain issuance
- Invoke VeritasID Soroban smart contracts to mint, revoke, and query on-chain credentials via the Stellar RPC
- Expose a versioned REST API and GraphQL interface for frontend consumption and third-party protocol integrations
- Maintain a credential index database that mirrors on-chain state for performant off-chain queries without requiring direct RPC calls for every read
- Implement a reputation scoring engine that computes weighted developer reputation scores from credential history
- Provide an organisation and DAO management API for configuring contribution-gated access rules
- Support webhook ingestion from GitHub to trigger automatic credential eligibility checks on PR merge events
- Emit structured logs, health check endpoints, and OpenTelemetry traces for full production observability

---

## Tech Stack

### Core Runtime & Framework

| Technology | Version | Purpose |
|---|---|---|
| [Node.js](https://nodejs.org) | 20.x LTS | Runtime environment |
| [TypeScript](https://www.typescriptlang.org) | 5.x | Type safety across the codebase |
| [Express.js](https://expressjs.com) | 4.x | HTTP server and REST routing |
| [Apollo Server](https://www.apollographql.com/docs/apollo-server) | 4.x | GraphQL API layer |

### Database & ORM

| Technology | Version | Purpose |
|---|---|---|
| [PostgreSQL](https://www.postgresql.org) | 16.x | Primary relational data store |
| [Prisma](https://www.prisma.io) | 5.x | Type-safe ORM and schema migration management |
| [Redis](https://redis.io) | 7.x | Session caching, rate limiting, and job queue backend |

### Background Jobs & Queues

| Technology | Version | Purpose |
|---|---|---|
| [BullMQ](https://bullmq.io) | 5.x | Async credential issuance and verification job queue |
| [node-cron](https://github.com/node-cron/node-cron) | 3.x | Scheduled on-chain state sync and reputation recomputation |

### GitHub Integration

| Technology | Purpose |
|---|---|
| [Octokit REST](https://github.com/octokit/rest.js) | GitHub REST API v3 — contribution and PR verification |
| [Octokit GraphQL](https://github.com/octokit/graphql.js) | GitHub GraphQL API v4 — contributor history queries |
| GitHub Webhooks | Real-time PR merge event ingestion for auto-issuance triggers |

### Credential Standards

| Technology | Purpose |
|---|---|
| [W3C VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0) | Verifiable Credential payload construction and validation |
| [DID Core](https://www.w3.org/TR/did-core) | Decentralised identifier generation for contributor profiles |
| [did:stellar method](https://github.com/stellar/stellar-did-method) | Stellar-native DID resolution |

### Blockchain Integration

| Technology | Version | Purpose |
|---|---|---|
| [Stellar SDK (JS)](https://stellar.github.io/js-stellar-sdk) | 12.x | Soroban contract invocation and Stellar network interaction |
| [Stellar Horizon](https://developers.stellar.org/network/horizon) | — | Transaction history queries and account monitoring |

### Authentication & Security

| Technology | Purpose |
|---|---|
| GitHub OAuth 2.0 | Developer identity authentication |
| [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) | JWT generation and session verification |
| [helmet](https://helmetjs.github.io) | HTTP security header hardening |
| [express-rate-limit](https://github.com/express-rate-limit/express-rate-limit) | Per-route API rate limiting |
| [zod](https://zod.dev) | Request body and parameter schema validation |

### Testing

| Technology | Version | Purpose |
|---|---|---|
| [Jest](https://jestjs.io) | 29.x | Unit and integration testing |
| [Supertest](https://github.com/ladjs/supertest) | 6.x | HTTP endpoint integration tests |
| [testcontainers](https://testcontainers.com/guides/nodejs-integration-testing) | 10.x | Ephemeral PostgreSQL and Redis for isolated test runs |
| [nock](https://github.com/nock/nock) | 13.x | GitHub API HTTP mocking |

### Observability & Tooling

| Technology | Purpose |
|---|---|
| [Pino](https://getpino.io) | Structured JSON logging |
| [OpenTelemetry](https://opentelemetry.io) | Distributed tracing and metrics instrumentation |
| [prom-client](https://github.com/siimon/prom-client) | Prometheus metrics exposure |
| ESLint + Prettier | Code linting and formatting |
| Husky + lint-staged | Pre-commit quality hooks |
| GitHub Actions | CI/CD pipeline |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     VeritasID Backend                        │
│                                                              │
│  ┌────────────────┐    ┌──────────────────┐                 │
│  │   REST API     │    │   GraphQL API    │                 │
│  │   (Express)    │    │   (Apollo)       │                 │
│  └───────┬────────┘    └────────┬─────────┘                 │
│          │                      │                            │
│          └──────────┬───────────┘                            │
│                     │                                        │
│          ┌──────────▼───────────┐                            │
│          │    Service Layer     │                            │
│          │                      │                            │
│          │ • AuthService        │                            │
│          │ • VerificationService│                            │
│          │ • IssuanceService    │                            │
│          │ • ReputationService  │                            │
│          │ • OrganisationService│                            │
│          └──────────┬───────────┘                            │
│                     │                                        │
│    ┌────────────────┼─────────────────┐                      │
│    │                │                 │                      │
│  ┌─▼────────┐  ┌────▼────┐  ┌────────▼──┐                  │
│  │PostgreSQL│  │  Redis  │  │  BullMQ   │                  │
│  │ (Prisma) │  │ (Cache) │  │ (Queues)  │                  │
│  └──────────┘  └─────────┘  └─────┬─────┘                  │
│                                    │                         │
│                     ┌──────────────▼──────────────┐         │
│                     │       External Services      │         │
│                     │  GitHub API │ Stellar/Soroban│         │
│                     └─────────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
veritasid-backend/
├── prisma/
│   ├── schema.prisma               # Database schema
│   └── migrations/                 # Prisma migration history
├── src/
│   ├── api/
│   │   ├── rest/
│   │   │   ├── routes/             # Express route definitions
│   │   │   └── middleware/         # Auth, rate limiting, validation middleware
│   │   └── graphql/
│   │       ├── schema/             # GraphQL type definitions
│   │       └── resolvers/          # GraphQL resolvers
│   ├── services/
│   │   ├── auth/                   # GitHub OAuth, JWT, wallet linking
│   │   ├── verification/           # GitHub contribution verification pipeline
│   │   ├── issuance/               # W3C VC construction and Soroban issuance
│   │   ├── reputation/             # Reputation score computation engine
│   │   ├── organisations/          # DAO and employer gating management
│   │   └── stellar/                # Stellar SDK and Soroban contract client
│   ├── jobs/
│   │   ├── workers/                # BullMQ worker definitions
│   │   └── schedulers/             # node-cron scheduled sync jobs
│   ├── webhooks/
│   │   └── github.ts               # GitHub webhook event handler
│   ├── lib/
│   │   ├── prisma.ts               # Prisma client singleton
│   │   ├── redis.ts                # Redis client singleton
│   │   └── logger.ts               # Pino structured logger
│   ├── types/                      # Global TypeScript types and interfaces
│   ├── config/                     # Environment config and constants
│   └── server.ts                   # Express + Apollo server entry point
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── .env.example
├── .eslintrc.cjs
├── jest.config.ts
├── tsconfig.json
└── package.json
```

---

## Getting Started

### Prerequisites

- Node.js >= 20.x
- npm >= 10.x or pnpm >= 9.x
- PostgreSQL 16.x instance
- Redis 7.x instance
- GitHub OAuth App credentials
- GitHub Personal Access Token (for API ingestion)
- Docker (optional — for local Postgres and Redis)

### Installation

```bash
# Clone the repository
git clone https://github.com/your-org/veritasid-backend.git
cd veritasid-backend

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env

# Start supporting services (optional, requires Docker)
docker-compose up -d postgres redis

# Run database migrations
npx prisma migrate dev

# Start the development server
npm run dev
```

The API will be available at `http://localhost:4000`.
GraphQL Playground: `http://localhost:4000/graphql`

---

## Environment Variables

```env
# Application
NODE_ENV=development
PORT=4000
APP_URL=http://localhost:4000

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/veritasid

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your_jwt_secret_minimum_32_characters
JWT_EXPIRES_IN=7d

# GitHub OAuth
GITHUB_CLIENT_ID=your_github_oauth_client_id
GITHUB_CLIENT_SECRET=your_github_oauth_client_secret
GITHUB_CALLBACK_URL=http://localhost:4000/api/v1/auth/github/callback
GITHUB_PAT=your_personal_access_token_for_api_ingestion
GITHUB_WEBHOOK_SECRET=your_github_webhook_secret

# Stellar
STELLAR_NETWORK=testnet                                  # testnet | mainnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
VERITASID_ISSUER_SECRET_KEY=S...                         # Issuer keypair (never commit)
CREDENTIAL_REGISTRY_CONTRACT_ID=your_contract_id
ACCESS_CONTROL_CONTRACT_ID=your_contract_id

# Frontend
FRONTEND_URL=http://localhost:5173
```

---

## API Reference

### REST Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Service health check |
| `GET` | `/api/v1/auth/github` | Initiate GitHub OAuth flow |
| `GET` | `/api/v1/auth/github/callback` | GitHub OAuth callback handler |
| `POST` | `/api/v1/auth/link-wallet` | Link Stellar wallet to GitHub identity |
| `GET` | `/api/v1/credentials/:address` | Get all credentials for a Stellar address |
| `GET` | `/api/v1/credentials/verify/:id` | Publicly verify a credential by ID |
| `POST` | `/api/v1/credentials/request` | Request credential issuance for a contribution |
| `GET` | `/api/v1/reputation/:address` | Get computed reputation score for an address |
| `GET` | `/api/v1/profile/:username` | Get public contributor profile |
| `POST` | `/api/v1/organisations` | Register an organisation for gated access |
| `GET` | `/api/v1/organisations/:id/verify/:address` | Verify a contributor against org gating rules |
| `POST` | `/webhook/github` | GitHub webhook event receiver |

### GraphQL

The GraphQL schema and introspection are available at `/graphql` in development. Core query types include `Credential`, `Contributor`, `ReputationScore`, `Organisation`, and `VerificationResult`.

---

## Available Scripts

```bash
npm run dev              # Start dev server with ts-node and hot reload
npm run build            # Compile TypeScript to dist/
npm run start            # Run compiled production build
npm run test             # Run Jest test suite
npm run test:watch       # Run Jest in watch mode
npm run test:coverage    # Generate coverage report
npm run lint             # Run ESLint
npm run format           # Run Prettier
npm run type-check       # TypeScript compiler check (no emit)
npm run db:migrate       # Run Prisma migrations
npm run db:studio        # Open Prisma Studio
npm run db:seed          # Seed database with development fixtures
```

---

## Contributing

Contributions are welcome. Please read the root-level [CONTRIBUTING.md](../CONTRIBUTING.md). All PRs require passing CI and at least one reviewer approval. Branch naming convention: `feat/`, `fix/`, `chore/`, `docs/`.

---

## License

[Apache 2.0](../LICENSE)
