# Lab 04 — Routing, REST Conventions & HTTP Status Codes

> **Phase 1 — NestJS foundations.**

---

## Objective

After this lab you can:

- Implement a full in-memory User CRUD with correct REST routes and HTTP status codes
- Use `@Param`, `@Body`, `@Query`, `@HttpCode` decorators
- Explain REST conventions: which verb + URL + status code maps to which operation

---

## Why

- REST gives predictable URLs and verbs so any client knows how to use your API without reading docs.
- Status codes are part of the contract — they tell the client *what happened*, not just the response body.

### REST conventions at a glance

| Operation | Method | URL | Success code | Notes |
|---|---|---|---|---|
| List all | `GET` | `/users` | `200` | Returns array |
| Get one | `GET` | `/users/:id` | `200` | `404` if not found |
| Create | `POST` | `/users` | `201` | Returns created resource |
| Update | `PATCH` | `/users/:id` | `200` | Partial update; `404` if not found |
| Delete | `DELETE` | `/users/:id` | `204` | No body in response |

> Use `PATCH` (partial update) rather than `PUT` (full replace) — it's more practical for most APIs.

---

## Libraries to install (and why)

No new dependencies — uses built-in `@nestjs/common` decorators.

---

## Steps

### 1. Extend the service with `update` and `remove`

In `src/users/users.service.ts`, add the two missing methods:

```typescript
import { Injectable } from '@nestjs/common';
import { randomUUID } from 'crypto';

export interface User {
  id: string;
  email: string;
  name: string;
}

@Injectable()
export class UsersService {
  private users: User[] = [];

  findAll(): User[] {
    return this.users;
  }

  findOne(id: string): User | undefined {
    return this.users.find(u => u.id === id);
  }

  create(data: { email: string; name: string }): User {
    const user: User = { id: randomUUID(), ...data };
    this.users.push(user);
    return user;
  }

  update(id: string, data: Partial<{ email: string; name: string }>): User | undefined {
    const user = this.findOne(id);
    if (!user) return undefined;
    Object.assign(user, data);
    return user;
  }

  remove(id: string): boolean {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return false;
    this.users.splice(index, 1);
    return true;
  }
}
```

### 2. Implement all five routes in the controller

Replace `src/users/users.controller.ts` with:

```typescript
import {
  Body, Controller, Delete, Get, HttpCode,
  Param, Patch, Post, Query,
} from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // GET /users?search=an
  @Get()
  findAll(@Query('search') search?: string) {
    const users = this.usersService.findAll();
    if (!search) return users;
    return users.filter(u =>
      u.name.toLowerCase().includes(search.toLowerCase()),
    );
  }

  // GET /users/:id
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);  // returns undefined if not found — fixed in Lab 07
  }

  // POST /users → 201
  @Post()
  create(@Body() body: { email: string; name: string }) {
    return this.usersService.create(body);
  }

  // PATCH /users/:id → 200
  @Patch(':id')
  update(
    @Param('id') id: string,
    @Body() body: Partial<{ email: string; name: string }>,
  ) {
    return this.usersService.update(id, body);
  }

  // DELETE /users/:id → 204 (no body)
  @Delete(':id')
  @HttpCode(204)
  remove(@Param('id') id: string) {
    this.usersService.remove(id);
  }
}
```

**Key points:**
- `@HttpCode(204)` overrides Nest's default `200` for DELETE — `204 No Content` is the REST convention.
- `@Query('search')` reads `?search=...` from the URL — the first taste of filtering, expanded in Lab 09.
- `PATCH` body is `Partial<...>` — the client only needs to send the fields it wants to change.

### 3. Test with a `.http` file

Create `requests/lab-04.http`:

```http
### Create user An
POST http://localhost:3000/users
Content-Type: application/json

{ "email": "an@example.com", "name": "An Nguyen" }

### Create user Binh
POST http://localhost:3000/users
Content-Type: application/json

{ "email": "binh@example.com", "name": "Binh Tran" }

### List all users
GET http://localhost:3000/users

### Search by name
GET http://localhost:3000/users?search=an

### Get one user — replace <id> with id from POST response
GET http://localhost:3000/users/<id>

### Update a user
PATCH http://localhost:3000/users/<id>
Content-Type: application/json

{ "name": "An Nguyen Updated" }

### Delete a user — expect 204 with no body
DELETE http://localhost:3000/users/<id>

### List again — deleted user should be gone
GET http://localhost:3000/users
```

For each request, **check the status code** in the response panel, not just the body.

---

## Acceptance criteria

- [ ] All five CRUD endpoints work against the in-memory store.
- [ ] `POST /users` returns `201`, `DELETE /users/:id` returns `204` with no body.
- [ ] `GET /users?search=an` filters the result correctly.
- [ ] You can explain the REST convention for each route (verb + URL + status code).
- [ ] No 404 handling yet — that comes in Lab 07.

---

## Git commit

```bash
git add .
git commit -m "lab-04: in-memory user CRUD with REST routes"
```

---

## References

- NestJS docs — Controllers (routing, params, status codes)
- MDN — HTTP response status codes

> Next: [lab-05](lab-05.md) — TypeORM setup & database connection.
