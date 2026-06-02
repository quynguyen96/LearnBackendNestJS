# Lab 01 — Database Fundamentals Refresher

> **Phase 0.** No application code yet. This lab solidifies the database knowledge every backend
> developer leans on daily. You'll think in tables and relationships, then practice with **raw SQL**
> against a real PostgreSQL instance. Everything you do later with TypeORM is just a typed wrapper
> over what you learn here.

---

## Objective

After this lab you can:

- Explain the main **types of databases** and when to use each
- Model the three core **relationships**: one-to-one (1-1), one-to-many (1-N), many-to-many (N-N)
- Choose **primary keys / foreign keys** and understand referential integrity
- Explain what an **index** is, why it speeds reads, and what it costs
- Write and reason about **JOINs**: INNER, LEFT, RIGHT, FULL OUTER
- Apply basic **normalization** to avoid duplicated data

---

## Why this matters (for a frontend developer)

As a frontend dev you consumed data the API handed you. On the backend, *you* design how that data
is stored. Get the data model wrong and every feature on top becomes painful: slow queries,
duplicated data, impossible reports. A solid relational model is the foundation the rest of this
course stands on.

---

## Part A — Types of databases

There is no single "best" database — each shape fits different problems.

| Type | Examples | Data shape | Best for |
|---|---|---|---|
| **Relational (SQL)** | PostgreSQL, MySQL, SQL Server | Tables with rows/columns, strict schema, relations | Transactional apps where data is structured and consistency matters (most business apps) |
| **Document (NoSQL)** | MongoDB, CouchDB | JSON-like documents, flexible schema | Rapidly changing / nested data, content stores |
| **Key-Value** | Redis, DynamoDB | `key → value` | Caching, sessions, counters, very fast lookups |
| **Wide-Column** | Cassandra, ScyllaDB | Rows with dynamic columns, partitioned | Huge write volumes, time-series at scale |
| **Graph** | Neo4j, Neptune | Nodes + edges | Highly connected data: social graphs, recommendations |
| **Search** | Elasticsearch, OpenSearch | Inverted index | Full-text search, log analytics |

### SQL vs NoSQL at a glance

- **SQL (relational):** fixed schema, strong consistency, powerful JOINs and transactions, vertical scaling first. Great when relationships and integrity matter.
- **NoSQL:** flexible schema, often eventual consistency, scales horizontally easily, weak/no JOINs. Great for scale, flexible shapes, or simple access patterns.

### Why PostgreSQL for this project

Our app manages users, groups and memberships — clearly **relational** data with integrity rules
("a membership must reference a real user and a real group"). PostgreSQL gives us strict schemas,
transactions, rich JOINs, and excellent SQL support. It's the default choice for transactional
business backends, and TypeORM speaks to it natively.

> Note: real systems often combine databases — e.g. PostgreSQL as the source of truth **plus** Redis
> for caching/sessions. We do exactly that later (Lab 21).

---

## Part B — Core relational concepts

- **Table** — a collection of rows of the same shape (e.g. `users`).
- **Row (record)** — one entity instance (one user).
- **Column (field)** — one attribute, with a **data type** (`uuid`, `varchar`, `int`, `boolean`, `timestamptz`, `jsonb`, …).
- **Primary key (PK)** — a column (or set) that **uniquely identifies** a row. We'll use `uuid`.
- **Foreign key (FK)** — a column that **references** a PK in another table; enforces **referential integrity** (you can't reference a user that doesn't exist).
- **Constraints** — rules the DB enforces: `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`.

---

## Part C — Relationships

The whole point of a relational database is *relationships between tables*.

### One-to-One (1-1)

One row in A relates to exactly one row in B.
Example: a `user` has one `user_profile` (extended details kept in a separate table).

```
users (1) ───── (1) user_profiles
```
Implemented with a **FK that is also UNIQUE** on the child table (`user_profiles.user_id UNIQUE`).
Use it to split rarely-used columns off a hot table, or for optional 1-1 extensions.

### One-to-Many (1-N)

One row in A relates to many rows in B; each B belongs to one A.
Example: one `user` **owns** many `groups`; each group has one owner.

```
users (1) ───< (N) groups        groups.owner_id → users.id
```
The **FK lives on the "many" side** (`groups.owner_id`).

### Many-to-Many (N-N)

Many rows in A relate to many rows in B. Requires a **join (junction) table**.
Example: a `user` is a **member** of many `groups`, and a `group` has many member users.

```
users (N) ───< user_groups >─── (N) groups
              user_id  group_id   (composite PK)
```
The join table `user_groups` holds one row per membership. This is exactly the
**"add user to group"** feature you'll build in Lab 13.

> Rule of thumb: 1-N → FK on the many side. N-N → a join table.

---

## Part D — Indexes

An **index** is a separate data structure (usually a B-tree) the database keeps so it can find rows
**without scanning the whole table**.

- **Why:** dramatically speeds up `WHERE`, `JOIN`, and `ORDER BY` on the indexed column(s).
- **Cost:** indexes take disk space and slow down `INSERT`/`UPDATE`/`DELETE` (the index must be maintained). So index deliberately, not everywhere.
- **Index these:** primary keys (automatic), foreign keys, and columns you frequently filter/sort/search by (e.g. `users.email`, `users.name` for search).
- **Unique index:** enforces uniqueness *and* speeds lookups (e.g. `users.email`).

You inspect whether a query uses an index with `EXPLAIN ANALYZE`:
- A **Seq Scan** = full table scan (often fine for tiny tables, bad for big ones on filtered queries).
- An **Index Scan** = the index was used.

---

## Part E — JOINs

JOINs combine rows from multiple tables. Picture two tables, **A (left)** and **B (right)**.

| JOIN | Returns |
|---|---|
| **INNER JOIN** | Only rows where there's a match in **both** A and B |
| **LEFT (OUTER) JOIN** | **All** rows from A; matching B columns, or `NULL` where no match |
| **RIGHT (OUTER) JOIN** | **All** rows from B; matching A columns, or `NULL` where no match |
| **FULL OUTER JOIN** | All rows from both; `NULL` on the side with no match |
| **CROSS JOIN** | Every combination (Cartesian product) — rarely what you want |
| **Self JOIN** | A table joined to itself (e.g. employee → manager) |

Mental model:
- **INNER** = the intersection (matches only).
- **LEFT** = "give me every A, and tell me about B if it exists" — the most common in app code.
- **RIGHT** = mirror of LEFT (often you just rewrite as a LEFT).
- **FULL** = everything from both, matched where possible.

You'll run all of these against real data in the steps below.

---

## Part F — Normalization (just enough)

Normalization = organizing data to **avoid duplication and update anomalies**.

- **1NF:** each column holds a single value (no comma-separated lists in a column).
- **2NF/3NF:** every non-key column depends on the *whole* key and nothing but the key — i.e. don't store data that belongs in another table.

Practical rule: **don't duplicate data; reference it.** Instead of storing a group's name on every
membership row, store the `group_id` and keep the name once in `groups`. (You can denormalize later
for performance, but start normalized.)

---

## Libraries / tools to install (and why)

This lab uses no app dependencies — only infrastructure:

- **Docker + Docker Compose** — to run PostgreSQL locally without a manual install. The project's
  `docker-compose.yml` (at the repo root) already defines a `postgres` service.
  Download: [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- **A SQL client** — [Navicat Premium Lite 17](https://www.navicat.com/en/download/navicat-premium-lite).
  *Why:* you want to see tables and run queries interactively.

---

## Steps

### 1. Start PostgreSQL

Open a terminal, `cd` into the project root (where `docker-compose.yml` lives), then run:

```bash
docker compose up -d postgres
docker compose ps          # confirm the postgres container is healthy
```

> **Note:** You must run these commands from the directory that contains `docker-compose.yml`, otherwise you will get the error `no configuration file provided: not found`.

### 2. Open a SQL session

Open **Navicat Premium Lite** and create a new PostgreSQL connection with the following settings:

| Field    | Value            |
|----------|------------------|
| Host     | `localhost`      |
| Port     | `5432`           |
| Database | `user_management`|
| Username | `app`            |
| Password | `app_password`   |

Click **Test Connection** to verify, then **Save** and **Open** the connection.
From there, open a new **Query** tab and run the SQL in the steps below.

### 3. Create the schema (run this SQL)

```sql
-- 1-N: a user owns many groups
CREATE TABLE users (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email      varchar(255) NOT NULL UNIQUE,
  name       varchar(120) NOT NULL,
  status     varchar(20)  NOT NULL DEFAULT 'active',
  created_at timestamptz  NOT NULL DEFAULT now()
);

CREATE TABLE groups (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        varchar(120) NOT NULL UNIQUE,
  description text,
  owner_id    uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT, -- FK = 1-N
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- N-N: membership join table
CREATE TABLE user_groups (
  user_id   uuid NOT NULL REFERENCES users(id)  ON DELETE CASCADE,
  group_id  uuid NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  joined_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, group_id)           -- composite PK: one membership per pair
);
```

### 4. Insert sample data

```sql
INSERT INTO users (email, name) VALUES
  ('an@example.com',   'An Nguyen'),
  ('binh@example.com', 'Binh Tran'),
  ('chi@example.com',  'Chi Le');

-- An owns a group
INSERT INTO groups (name, description, owner_id)
SELECT 'Engineering', 'Eng team', id FROM users WHERE email = 'an@example.com';

-- memberships: An and Binh are in Engineering; Chi is in none
INSERT INTO user_groups (user_id, group_id)
SELECT u.id, g.id
FROM users u, groups g
WHERE g.name = 'Engineering' AND u.email IN ('an@example.com','binh@example.com');
```

### 5. Practice JOINs

```sql
-- INNER JOIN: only users who ARE in a group
SELECT u.name, g.name AS group_name
FROM users u
INNER JOIN user_groups ug ON ug.user_id = u.id
INNER JOIN groups g       ON g.id = ug.group_id;

-- LEFT JOIN: ALL users, with their group if any (Chi shows with NULL group)
SELECT u.name, g.name AS group_name
FROM users u
LEFT JOIN user_groups ug ON ug.user_id = u.id
LEFT JOIN groups g       ON g.id = ug.group_id;

-- "Who is NOT in any group?" — LEFT JOIN + IS NULL
SELECT u.name
FROM users u
LEFT JOIN user_groups ug ON ug.user_id = u.id
WHERE ug.user_id IS NULL;

-- RIGHT JOIN: all groups, with members if any
SELECT g.name AS group_name, u.name
FROM users u
RIGHT JOIN user_groups ug ON ug.user_id = u.id
RIGHT JOIN groups g       ON g.id = ug.group_id;

-- 1-N: each group with its owner's name
SELECT g.name AS group_name, owner.name AS owner
FROM groups g
INNER JOIN users owner ON owner.id = g.owner_id;
```

For each query, **predict the result before running it**, then check.

### 6. Indexes & EXPLAIN

```sql
-- search users by name (case-insensitive) — common in our app
EXPLAIN ANALYZE
SELECT * FROM users WHERE name ILIKE '%an%';        -- likely a Seq Scan on small data

-- add an index on email (already unique = already indexed); add one for name search
CREATE INDEX idx_users_name ON users (name);

EXPLAIN ANALYZE
SELECT * FROM users WHERE name = 'An Nguyen';        -- should use Index Scan
```

Note: a plain B-tree index does **not** help `ILIKE '%term%'` (leading wildcard). That's a hint that
free-text search may later need a trigram index (`pg_trgm`) or a search engine — good to know exists,
not needed now.

### 7. (Optional) Model a 1-1

```sql
CREATE TABLE user_profiles (
  user_id uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE, -- PK = FK = UNIQUE => 1-1
  bio     text,
  phone   varchar(30)
);
```
Notice the PK *is* the FK — that's what makes it one-to-one.

---

## Acceptance criteria

- [ ] PostgreSQL is running in Docker and you can connect to it.
- [ ] You created `users`, `groups`, `user_groups` with correct PK/FK constraints.
- [ ] You can explain why `user_groups` needs a composite primary key.
- [ ] You ran INNER / LEFT / RIGHT joins and can explain how each differs.
- [ ] You found "users not in any group" using a LEFT JOIN + `IS NULL`.
- [ ] You created an index and saw an Index Scan in `EXPLAIN ANALYZE`.
- [ ] You can describe, in your own words, 1-1 vs 1-N vs N-N and how each is stored.

---

## References

- PostgreSQL docs — Data Types, Constraints, Indexes, `EXPLAIN`
- PostgreSQL docs — `SELECT` / JOIN syntax
- "Database normalization" — any 1NF/2NF/3NF primer

> Next: [lab-02](lab-02.md) — bootstrap the NestJS project.
