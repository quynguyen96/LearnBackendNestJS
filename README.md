# User Management API — Backend Learning Project

A hands-on learning project for a **frontend (Next.js / TypeScript) developer** moving into backend.
You build one real application — a **User Management API** — across 30 guided labs, learning
**NestJS + PostgreSQL + TypeORM** by shipping features, not by watching tutorials.

---

## 1. Purpose

By the end of this project you should be able to **independently pick up and ship a real backend task**:
model data, write migrations, build authenticated and authorized endpoints, paginate/search/filter,
wrap operations in transactions, test, document, and ship with Docker.

The product you build is an **admin backend for managing users**:

- Create / edit / delete users (CRUD)
- List users with **pagination + search + filtering**
- Organize users into **groups** (add / remove members)
- Authentication (register / login with JWT)
- Authorization (admin vs user roles, resource ownership)
- Roles & permissions (capstone)

> You start each lab from where the previous one ended. The repo grows with you.

---

## 2. Prerequisites

You should already be comfortable with:

- TypeScript, `async/await`, JSON
- HTTP basics (methods, status codes) and the REST mindset
- npm and the terminal
- Git

You should have installed:

- **Node.js 20+** and npm
- **Docker** + Docker Compose (for PostgreSQL and Redis)
- A code editor (VS Code recommended) with the **REST Client** extension (or Postman)
- A SQL GUI client: **Navicat Premium Lite 17** (optional but very helpful)

---

## 3. Tech stack & why each piece

| Tool | Role | Why it's here |
|---|---|---|
| **NestJS** | Backend framework | Opinionated structure + dependency injection; feels like a server-side Angular, great for TypeScript devs |
| **TypeScript** | Language | Type safety end-to-end; you already know it |
| **PostgreSQL** | Relational database | Robust, standards-compliant SQL DB; the default choice for transactional apps |
| **TypeORM** | ORM | Maps classes ↔ tables; gives you the Repository pattern, migrations, relations |
| **Docker / Compose** | Local infrastructure | Run Postgres + Redis with one command; no manual installs |
| **class-validator / class-transformer** | Validation & serialization | Validate incoming DTOs; hide sensitive fields in responses |
| **@nestjs/config + Joi** | Configuration | Env-based config, validated at boot (fail fast) |
| **bcrypt** | Password hashing | Never store plaintext passwords |
| **@nestjs/jwt + Passport** | Authentication | Stateless JWT auth |
| **@nestjs/throttler** | Rate limiting | Protect endpoints (e.g. login) from abuse |
| **@nestjs/cache-manager (+ Redis)** | Caching | Reduce DB load on hot reads |
| **@nestjs/swagger** | API docs | Interactive OpenAPI documentation (set up early, in Lab 02) |
| **Nest Logger** | Logging | Built-in logging, set up early so you instrument as you build |
| **@nestjs/schedule / BullMQ** | Jobs | Cron jobs and background queues |

> Libraries are installed **incrementally, inside the lab that needs them** — each lab explains
> *what* you install and *why*. Do not pre-install everything.

---

## 4. Architecture

This project uses **NestJS's idiomatic feature-module architecture** — a pragmatic layered approach:

```
HTTP request
   │
   ▼
Controller   ── presentation layer: parse request, return response, status codes
   │
   ▼
Service      ── business logic layer: rules, orchestration, transactions
   │
   ▼
Repository   ── data access layer: TypeORM Repository<Entity>
   │
   ▼
PostgreSQL
```

Each **feature** (users, groups, auth) is a self-contained **Module** that wires its own
controller + service + entities together via **Dependency Injection**.

> **Coming from .NET Clean Architecture?** Nest does not enforce Clean Architecture by default,
> but it maps cleanly onto it if you want (Domain / Application / Infrastructure / Presentation),
> because Nest's DI container is as capable as .NET's. The Repository pattern is built into TypeORM
> (`Repository<T>`), and the Unit of Work maps onto a TypeORM transaction's shared `EntityManager`.
> This project intentionally starts with the simpler feature-module style so you learn the framework
> first; refactoring to Clean Architecture later is a good exercise.

---

## 5. Source code structure (target)

You build toward this layout (it doesn't all exist on day one):

```
LearnBackendNestJS/
├── docs/                      # the 30 labs (lab-01.md … lab-30.md)
├── src/
│   ├── main.ts                # app bootstrap
│   ├── app.module.ts          # root module
│   ├── config/                # env config + validation
│   ├── common/                # cross-cutting: filters, interceptors, decorators, pagination
│   │   ├── filters/
│   │   ├── interceptors/
│   │   ├── decorators/
│   │   └── dto/
│   ├── database/
│   │   ├── data-source.ts     # TypeORM CLI DataSource (for migrations)
│   │   └── migrations/
│   └── modules/
│       ├── users/
│       │   ├── users.module.ts
│       │   ├── users.controller.ts
│       │   ├── users.service.ts
│       │   ├── entities/user.entity.ts
│       │   └── dto/
│       ├── groups/
│       └── auth/
├── test/                      # e2e tests
├── docker-compose.yml         # postgres + redis
├── .env.example
├── .gitignore
└── package.json
```

---

## 6. How to use this project

1. Read the labs **in order**, one at a time, in `docs/`.
2. Each lab has the same shape:
   - **Objective** — what you'll be able to do after it
   - **Why** — the reasoning behind the concepts
   - **Libraries to install (and why)**
   - **Steps** — do these yourself; type the code, don't paste blindly
   - **Acceptance criteria** — how to know you're done
   - **Git commit** — commit your work
   - **References**
3. After each lab, **commit** with the suggested message.
4. If a lab takes two sessions, take two. Depth beats speed.

### Recommended pace

~2–3 hours/day → roughly **one lab per day, ~30 days**. Adjust freely.

---

## 7. Running the project (once you reach the relevant labs)

```bash
# 1. start infrastructure (Postgres + Redis)
docker compose up -d

# 2. install dependencies
npm install

# 3. copy env and fill values
cp .env.example .env

# 4. run database migrations
npm run migration:run

# 5. start the API in watch mode
npm run start:dev

# API:      http://localhost:3000
# Swagger:  http://localhost:3000/docs   (set up from Lab 02, deepened in Lab 24)
```

---

## 8. Labs index

See [`docs/README.md`](docs/README.md) for the full list and learning map.

Start here → [`docs/lab-01.md`](docs/lab-01.md) (Database fundamentals refresher).
