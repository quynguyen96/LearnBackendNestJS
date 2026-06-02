# Lab 02 — NestJS Setup & Project Bootstrap

> **Phase 1.** You'll scaffold the NestJS application and understand its structure before adding any
> features. No database yet — we focus on the framework.

---

## Objective

After this lab you can:

- Create a NestJS project with the CLI
- Explain the role of `main.ts`, `app.module.ts`, controllers and services
- Run the dev server and add a simple route
- Describe the request lifecycle at a high level

---

## Why

A framework gives you structure, dependency injection, and testability that raw Node/Express does
not. NestJS organizes code into **modules**, handles wiring for you, and is built for TypeScript —
which you already know. Learning its anatomy now makes every later lab easier.

**Request lifecycle (high level):**
```
request → middleware → guard → interceptor → pipe → controller handler → service → response
```
You'll meet each piece across the next labs.

---

## Libraries to install (and why)

- **Node.js 20+** (prerequisite) — the runtime. *Why:* NestJS runs on Node.
- **@nestjs/cli** (global) — scaffolding tool. *Why:* generates project + modules/controllers/services with correct boilerplate so you don't hand-write it.
- **@nestjs/swagger** — OpenAPI/Swagger integration. *Why:* we set up the docs UI **now** so that from the very first run you have a `/docs` page (like ASP.NET's built-in Swagger). It auto-discovers routes, so the UI fills in as you add endpoints. Deep annotation comes later in Lab 24.
- **nestjs-pino + pino-http + pino-pretty** — structured logger. *Why:* NestJS's built-in logger writes plain human-readable strings — fine for development but unusable in production where you need machine-readable JSON to filter, search and correlate logs. Pino writes async (non-blocking), structured JSON to stdout, and automatically logs every HTTP request/response. `pino-pretty` makes the output readable in your terminal during development.

```bash
node -v          # confirm 20+
npm i -g @nestjs/cli
# The packages below are installed inside the steps after `nest new` creates package.json
```

---

## Steps

### 1. Create the project

> You already have this repo (with `docs/`). Generate the Nest app **into the current folder** so it
> merges with the existing structure:

```bash
# from the repo root
nest new . --skip-git    # choose npm when prompted
```

If the CLI refuses because the folder is not empty, generate into a temporary folder instead:

```bash
nest new tmp-nest --skip-git   # choose npm
# then move these files/folders into the repo root:
#   src/  package.json  tsconfig.json  tsconfig.build.json  nest-cli.json  .eslintrc.js  .prettierrc
```

### 2. Install dependencies

```bash
npm install
```

### 3. Explore what was generated

Open and read every file — understand what each decorator does before moving on:

- `src/main.ts` — entry point; creates the Nest application and starts the HTTP listener.
- `src/app.module.ts` — the **root module**; the `@Module` decorator tells Nest which controllers and providers belong here. Every feature you add will be imported into this module (or into a child module that this one imports).
- `src/app.controller.ts` — an example controller. `@Controller()` marks the class as a route handler; `@Get()` maps a method to a GET route.
- `src/app.service.ts` — an example provider. `@Injectable()` marks it for Nest's **dependency injection** container — meaning Nest creates one instance and passes it wherever it is requested (e.g. into the controller's constructor).

### 4. Run it

```bash
npm run start:dev      # watch mode: restarts on file change
```
Open `http://localhost:3000` → you should see **"Hello World!"**.

### 5. Add your own route

In `src/app.controller.ts`, **add** the `ping()` method to the existing class (do not replace the existing `getHello()` method, just add below it):

```typescript
@Get('ping')
ping() {
  return { status: 'ok', time: new Date().toISOString() };
}
```

Visit `http://localhost:3000/ping` → you should get a JSON object. Notice Nest serialized the object
to JSON for you automatically.

### 6. Set up Swagger now (so you see it on first run)

Unlike ASP.NET, NestJS has **no fixed `/swagger/index.html` path** — you choose the path yourself.
The path is the first argument to `SwaggerModule.setup()`. Here we use `/docs`.

Install the package:

```bash
npm i @nestjs/swagger
```

Edit `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // --- Swagger setup ---
  const config = new DocumentBuilder()
    .setTitle('User Management API')
    .setDescription('Learning project — users, groups, auth')
    .setVersion('1.0')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document); // UI at /docs
  // ---------------------

  await app.listen(3000);
}
bootstrap();
```

(Optional, recommended) enable the Swagger CLI plugin so DTO properties are documented
automatically without writing `@ApiProperty` on every field. In `nest-cli.json`:

```json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

Restart `npm run start:dev` and open:

- **Swagger UI:** `http://localhost:3000/docs`
- **Raw OpenAPI JSON:** `http://localhost:3000/docs-json`

Right now it only shows `/` and `/ping` — that's expected. The UI grows automatically as you add
controllers in later labs. (Lab 24 deepens this: tags, request/response schemas, and bearer auth.)

> Want it to feel exactly like .NET? Use `SwaggerModule.setup('swagger', app, document)` →
> `http://localhost:3000/swagger`.

### 7. Set up structured logging with Pino

NestJS's built-in logger prints plain text strings — fine for development but not queryable in production. We replace it with **Pino**: the fastest Node.js logger, writing non-blocking async JSON to stdout, with automatic HTTP request/response logging.

Install the packages:

```bash
npm i nestjs-pino pino-http
npm i -D pino-pretty    # dev-only: pretty-prints JSON in your terminal
```

Create `src/logger/app-logger.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { LoggerModule } from 'nestjs-pino';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    LoggerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        const isDev = config.get('NODE_ENV') !== 'production';
        return {
          pinoHttp: {
            level: isDev ? 'debug' : 'info',
            transport: isDev
              ? { target: 'pino-pretty', options: { colorize: true, singleLine: true } }
              : undefined,                  // production: raw JSON to stdout
            redact: ['req.headers.authorization', 'req.headers.cookie'],
          },
        };
      },
    }),
  ],
})
export class AppLoggerModule {}
```

Import it in `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { AppLoggerModule } from './logger/app-logger.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    AppLoggerModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Update `src/main.ts` to hand bootstrap logs over to Pino as well (add `bufferLogs` and `useLogger`):

```typescript
import { NestFactory } from '@nestjs/core';
import { Logger } from 'nestjs-pino';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));   // replace Nest's default logger with Pino

  const config = new DocumentBuilder()
    .setTitle('User Management API')
    .setDescription('Learning project — users, groups, auth')
    .setVersion('1.0')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  await app.listen(3000);
}
bootstrap();
```

> **Why `bufferLogs: true`?** Without it, NestJS uses its own logger for the brief window between `NestFactory.create()` and `useLogger()`. `bufferLogs` holds those early messages and replays them through Pino once it is registered, so every log line — including bootstrap — is structured JSON.

Restart `npm run start:dev`. You should now see colorised, single-line log output in your terminal, and every HTTP request is logged automatically (method, URL, status code, response time).

To use the logger inside a service or controller, inject it:

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  getHello(): string {
    this.logger.log('getHello called');
    return 'Hello World!';
  }
}
```

### 8. Initialize Git

```bash
git init
git add .
git commit -m "lab-02: bootstrap NestJS project"
```

---

## Acceptance criteria

- [ ] `npm run start:dev` runs without errors.
- [ ] `GET /` returns "Hello World!".
- [ ] `GET /ping` returns your JSON object.
- [ ] **Swagger UI loads at `http://localhost:3000/docs`** and shows the `/ping` route.
- [ ] Terminal shows colorised Pino log lines; each HTTP request is logged automatically.
- [ ] You can explain what `main.ts` and `app.module.ts` each do, and that the Swagger path is whatever you pass to `SwaggerModule.setup()`.
- [ ] You can explain why Pino is preferred over the built-in NestJS logger in production.

---

## Git commit

```bash
git add .
git commit -m "lab-02: bootstrap NestJS project with Swagger and Pino logger"
```

---

## References

- NestJS docs — First steps, Controllers, Providers
- NestJS docs — OpenAPI (Swagger) — setup and the CLI plugin

> Next: [lab-03](lab-03.md) — Modules, Controllers, Providers & Dependency Injection.
