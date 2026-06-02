# Lab 03 — Modules, Controllers, Providers & Dependency Injection

> **Phase 1.** The single most important Nest concept: **Dependency Injection (DI)**. You'll create
> your first feature module — `users` — backed by an in-memory store (no database yet).

---

## Objective

After this lab you can:

- Create a feature module with `nest generate`
- Explain `controllers`, `providers`, `imports`, `exports` in `@Module`
- Inject a service into a controller via the constructor (DI)
- Keep controllers thin and put logic in services

---

## Why

- **Dependency Injection:** instead of a class building its own dependencies (`new SomethingService()`),
  it *declares* them in its constructor and Nest **supplies** them. This makes code testable (you can
  inject a fake in tests) and decoupled. If you've used .NET's `IServiceCollection`, this is the same idea.
- **Separation of concerns:** controllers handle HTTP (parse request, return response); services hold
  business logic. Keeping them separate keeps each simple.

---

## Libraries to install (and why)

None — everything needed ships with NestJS. We only use the CLI to generate files.

---

## Steps

### 1. Generate the users module

```bash
nest g module users
nest g controller users
nest g service users
```

This creates `src/users/` with `users.module.ts`, `users.controller.ts`, `users.service.ts` and **automatically** adds `UsersModule` to the `imports` array of `AppModule`. Open `src/app.module.ts` to confirm:

```typescript
@Module({
  imports: [AppLoggerModule, UsersModule],  // ← UsersModule added by the CLI
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### 2. Define a temporary User shape + in-memory store

In `users.service.ts`:

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
}
```

`@Injectable()` marks the class as a provider Nest can inject.

### 3. Inject the service into the controller

In `users.controller.ts`:

```typescript
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  // DI: Nest creates UsersService and passes it in. No `new` anywhere.
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() body: { email: string; name: string }) {
    return this.usersService.create(body);
  }
}
```

`@Param('id')` extracts the `:id` segment from the URL and passes it as a string argument.

### 4. Understand the module

Open `users.module.ts`:

```typescript
@Module({
  controllers: [UsersController],   // route handlers owned by this module
  providers: [UsersService],        // injectables available inside this module
  // exports: [UsersService],       // uncomment to expose UsersService to OTHER modules that import UsersModule
})
export class UsersModule {}
```

**`imports` vs `exports` explained:**

- **`providers`** — classes Nest can inject *within this module*. `UsersService` is listed here, so `UsersController` can receive it in its constructor.
- **`exports`** — a subset of `providers` you want to make available to *other* modules. If a future `GroupsModule` needs to call `UsersService`, it would `import: [UsersModule]` in its `@Module`, and `UsersModule` would need `exports: [UsersService]`. Without `exports`, the service is private to `UsersModule`.
- **`imports`** — other modules whose exported providers this module needs. For example, once we add TypeORM in Lab 05, we will `import: [TypeOrmModule.forFeature([User])]` here.

### 5. Test

Start the server:

```bash
npm run start:dev
```

Create a file `requests/lab-03.http` (requires the VS Code **REST Client** extension) and run each request:

```http
### Create a user
POST http://localhost:3000/users
Content-Type: application/json

{ "email": "an@example.com", "name": "An Nguyen" }

### List all users
GET http://localhost:3000/users

### Get one user — replace <id> with the id returned from POST
GET http://localhost:3000/users/<id>
```

Expected results:
- `POST /users` → `201 Created` with `{ id, email, name }`
- `GET /users` → array containing your new user
- `GET /users/:id` → the single user object (or `undefined` if id not found — we fix the 404 in Lab 04)

### 6. Verify Swagger

Open `http://localhost:3000/docs` — you should see the three new routes (`GET /users`, `POST /users`, `GET /users/{id}`) listed automatically. No extra annotation was needed — Swagger discovered them from the decorators.

---

## Acceptance criteria

- [ ] `GET /users`, `POST /users`, and `GET /users/:id` all work against the in-memory store.
- [ ] There is **no** `new UsersService()` anywhere — the service is injected via the constructor.
- [ ] You can explain what `providers`, `exports`, and `imports` do in a `@Module`.
- [ ] Swagger UI at `/docs` shows all three `/users` routes automatically.
- [ ] You verified that `AppModule.imports` was updated automatically by the CLI.

---

## Git commit

```bash
git add .
git commit -m "lab-03: users module with DI (in-memory)"
```

---

## References

- NestJS docs — Modules, Providers, Dependency Injection

> Next: [lab-04](lab-04.md) — Routing, REST conventions & HTTP status codes.
