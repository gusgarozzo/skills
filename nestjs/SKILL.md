# NestJS Clean Code + SOLID Engineering Skill

## Purpose

Use this skill whenever creating, reviewing, refactoring, or extending a NestJS + TypeScript backend.

The goal is production-oriented code that is:

- idiomatic NestJS
- strongly typed and easy to read
- cohesive and low-coupled
- testable without unnecessary mocking or indirection
- aligned with SOLID without overengineering
- resilient to unexpected application errors
- consistent across modules and features
- compatible with REST APIs and adaptable to other Nest transports

This file is the canonical source of truth for the skill. Editor/agent-specific instruction files should point to it or mirror its rules.

---

## 1. Non-negotiable engineering rules

1. Prefer clarity over cleverness.
2. Prefer simple code over abstractions introduced only to satisfy a pattern.
3. Never use `any` to silence TypeScript. Use a real type, `unknown`, a generic, or a discriminated union.
4. Treat external input as untrusted. Validate DTOs/schemas before application logic runs.
5. Keep controllers thin: transport concerns in controllers; business rules in application/domain services.
6. Do not put business logic inside DTOs, controllers, repositories, or database entities.
7. Do not expose persistence models directly as API contracts.
8. Make dependencies explicit through constructor injection.
9. Depend on abstractions at architectural boundaries when doing so provides a real testing or substitution benefit.
10. Catch exceptions only when you can recover, translate, add meaningful context, or enforce a boundary. Do not catch-and-rethrow unchanged.
11. Never expose stack traces, SQL errors, tokens, credentials, or internal implementation details to API clients.
12. Unexpected HTTP request errors must be handled by a global exception filter.
13. Never claim that an HTTP exception filter alone guarantees process survival. Process-level failures require graceful shutdown and a process supervisor/runtime such as Docker, Kubernetes, ECS, systemd, or another appropriate manager.
14. Every new behavior should have an appropriate test: unit tests for logic and integration/e2e tests for boundaries.
15. Do not refactor unrelated code while implementing a focused change.

---

## 2. TypeScript and syntax

### Compiler discipline

Prefer a strict TypeScript configuration. At minimum:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

Do not add every compiler flag blindly to an existing project. Preserve project compatibility and enable stricter options incrementally when necessary.

### Syntax rules

Use:

```ts
const user = await this.usersService.findById(userId);
```

Not:

```ts
let user;
user = await this.usersService.findById(userId);
```

Prefer early returns:

```ts
if (!user) {
  throw new NotFoundException('User not found');
}

return user;
```

Avoid deep nesting:

```ts
if (!condition) {
  return;
}

if (!otherCondition) {
  return;
}

// main path
```

Use descriptive names instead of comments explaining obvious code.

Bad:

```ts
// Get active users
const x = await repo.find({ where: { active: true } });
```

Good:

```ts
const activeUsers = await repo.find({ where: { active: true } });
```

### `unknown` over `any`

Use `unknown` at untrusted boundaries:

```ts
function normalizeError(error: unknown): Error {
  if (error instanceof Error) {
    return error;
  }

  return new Error('Unknown error');
}
```

Never do this:

```ts
function normalizeError(error: any): Error {
  return error;
}
```

### Async code

- Prefer `async`/`await` for application code.
- Do not create floating promises accidentally.
- Await I/O that must complete before continuing.
- Make fire-and-forget behavior explicit and observable.
- Do not swallow rejected promises.

---

## 3. NestJS architecture

Organize primarily by feature/domain capability, not by framework artifact alone.

Preferred:

```text
src/
  modules/
    users/
      application/
      domain/
      infrastructure/
      presentation/
      users.module.ts
```

For smaller projects, avoid unnecessary layers:

```text
users/
  users.controller.ts
  users.service.ts
  users.repository.ts
  dto/
  users.module.ts
```

### Controllers

Controllers should:

- parse/receive transport input
- delegate work
- return application results
- map transport-specific concerns

Controllers should not:

- contain business rules
- query the database directly
- perform complex transformations
- contain long `try/catch` blocks only to return HTTP errors

Example:

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  findById(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponse> {
    return this.usersService.findById(id);
  }
}
```

### Providers/services

Services should own application behavior and orchestration, not HTTP mechanics.

Good:

```ts
@Injectable()
export class CreateUserService {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly passwordHasher: PasswordHasher,
  ) {}

  async execute(input: CreateUserInput): Promise<User> {
    const existingUser = await this.usersRepository.findByEmail(input.email);

    if (existingUser) {
      throw new ConflictException('User already exists');
    }

    const passwordHash = await this.passwordHasher.hash(input.password);

    return this.usersRepository.create({
      email: input.email,
      passwordHash,
    });
  }
}
```

Avoid a 500-line `UsersService` containing every use case. Split by behavior when cohesion drops.

### Modules

Each module should have a clear responsibility.

- Export only what other modules genuinely need.
- Keep implementation details private where possible.
- Avoid circular dependencies.
- Avoid a giant shared module becoming a dumping ground.

---

## 4. SOLID applied pragmatically to NestJS

### S — Single Responsibility

A class should have one primary reason to change.

Bad:

```ts
UsersService {
  // validation
  // password hashing
  // database access
  // email sending
  // HTTP response formatting
}
```

Prefer cohesive responsibilities:

```text
CreateUserService
UsersRepository
PasswordHasher
UserMailer
UsersController
```

Do not split every three lines into a class. SRP is about responsibility, not class count.

### O — Open/Closed

Prefer stable interfaces at real extension points.

Example:

```ts
export interface PasswordHasher {
  hash(value: string): Promise<string>;
}
```

Implementations can be swapped without changing the use case.

Do not introduce an interface for every concrete class when there is no plausible alternative implementation or testing boundary.

### L — Liskov Substitution

Implementations must honor the behavior promised by their abstraction.

If `PaymentGateway.charge()` promises a rejected operation for a payment failure, an implementation should not silently return success with a different semantic meaning.

### I — Interface Segregation

Prefer small interfaces:

```ts
export interface UserReader {
  findById(id: string): Promise<User | null>;
}

export interface UserWriter {
  create(input: CreateUserRepositoryInput): Promise<User>;
}
```

Avoid one enormous interface that every implementation must depend on.

### D — Dependency Inversion

High-level application logic should depend on abstractions at meaningful boundaries.

```ts
@Injectable()
export class CreateOrderService {
  constructor(
    private readonly ordersRepository: OrdersRepository,
    private readonly paymentGateway: PaymentGateway,
  ) {}
}
```

Nest dependency injection should wire infrastructure implementations to those boundaries.

---

## 5. DTOs, validation, and serialization

Validate all external input before acting on it.

Recommended global setup for a class-validator based project:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

Rules:

- DTOs describe input contracts.
- Response DTOs describe output contracts when the API contract should differ from internal models.
- Never trust client-provided IDs, roles, ownership, prices, or permissions.
- Prefer explicit validation over manual scattered checks.
- Use `ParseUUIDPipe`, `ParseIntPipe`, etc. where appropriate.

Example:

```ts
export class CreateUserDto {
  @IsEmail()
  email!: string;

  @IsString()
  @MinLength(12)
  password!: string;
}
```

When the project uses Zod, Valibot, ArkType, or another Standard Schema implementation, use the project's established validation strategy consistently instead of mixing incompatible conventions unnecessarily.

---

## 6. Exceptions and error handling

### Use Nest HTTP exceptions intentionally

Choose the status that describes the client-visible condition:

```ts
throw new BadRequestException('Invalid input');
throw new UnauthorizedException('Authentication required');
throw new ForbiddenException('Access denied');
throw new NotFoundException('Resource not found');
throw new ConflictException('Resource already exists');
```

Do not use `InternalServerErrorException` for every failure.

### Domain/application errors

For larger systems, define stable machine-readable error codes instead of making clients parse human messages.

Example:

```ts
throw new ConflictException('User already exists', {
  errorCode: 'USER_ALREADY_EXISTS',
});
```

### Catch only at meaningful boundaries

Bad:

```ts
try {
  return await this.repository.save(entity);
} catch (error) {
  throw error;
}
```

Good:

```ts
try {
  return await this.paymentGateway.charge(input);
} catch (error) {
  this.logger.error('Payment provider failed', error);
  throw new PaymentProviderUnavailableException();
}
```

The catch adds behavior: logging/context and translation into a stable application error.

---

## 7. Global exception filter — mandatory baseline

Every HTTP NestJS service should have a global catch-all filter so unexpected request-level exceptions are converted into a controlled response and logged appropriately.

### Important reliability distinction

A global exception filter protects the HTTP request lifecycle. It does **not** guarantee that the Node.js process can never terminate.

The filter should handle unexpected exceptions that reach Nest's exception layer. Process-level failures such as an `uncaughtException`, a fatal runtime condition, OOM, container termination, or a failed startup require separate lifecycle/shutdown strategy and an external supervisor/runtime.

Do not write code that pretends the service is immortal.

### Recommended implementation

Prefer Nest's `BaseExceptionFilter` plus `HttpAdapterHost` when the filter is intended to be global and platform-aware.

```ts
import {
  ArgumentsHost,
  Catch,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { BaseExceptionFilter, HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  constructor(private readonly httpAdapterHost: HttpAdapterHost) {
    super(httpAdapterHost.httpAdapter);
  }

  override catch(exception: unknown, host: ArgumentsHost): void {
    const { httpAdapter } = this.httpAdapterHost;
    const context = host.switchToHttp();

    const request = context.getRequest<{ method: string; url: string }>();
    const response = context.getResponse();

    const isHttpException = exception instanceof HttpException;
    const status = isHttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = isHttpException
      ? exception.getResponse()
      : {
          statusCode: status,
          message: 'Internal server error',
        };

    this.logException(exception, request);

    httpAdapter.reply(response, responseBody, status);
  }

  private logException(
    exception: unknown,
    request: { method: string; url: string },
  ): void {
    if (exception instanceof Error) {
      this.logger.error(
        `${request.method} ${request.url} - ${exception.message}`,
        exception.stack,
      );
      return;
    }

    this.logger.error(
      `${request.method} ${request.url} - Unknown exception`,
    );
  }
}
```

### Global registration

When the filter has dependencies, register it through `APP_FILTER` so Nest can inject them:

```ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: AllExceptionsFilter,
    },
  ],
})
export class AppModule {}
```

For a filter with no DI dependencies, `app.useGlobalFilters(...)` is also valid. Prefer `APP_FILTER` when dependency injection is needed.

### Security requirements for the filter

Never return these to clients:

- stack traces
- database/ORM error messages
- SQL statements
- access tokens
- API keys
- passwords
- internal service URLs
- filesystem paths
- provider credentials

Logging must also avoid secrets and unnecessary personal data. Sanitize structured metadata before logging.

### Preserve expected HTTP errors

Do not turn every known `HttpException` into a generic 500. Known application/client errors should preserve their HTTP status and safe response body; unexpected errors should become 500.

### Response shape

A project may standardize a response envelope, for example:

```ts
{
  statusCode: 500,
  message: 'Internal server error',
  path: '/users/123',
  timestamp: '2026-09-24T17:00:00.000Z',
  requestId: '...'
}
```

Use one stable schema across the API. Do not expose implementation details merely to make debugging easier for the client.

---

## 8. Process resilience and graceful shutdown

A request-level filter is not a process supervisor.

For production services:

- enable graceful shutdown hooks where appropriate
- close database connections and external clients cleanly
- stop accepting new traffic during shutdown
- rely on the deployment/runtime to restart unhealthy processes
- implement health/readiness endpoints where the platform requires them
- monitor error rate, latency, restarts, and resource exhaustion

Example bootstrap setting:

```ts
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();
```

Do not blindly continue executing after a truly unsafe process-level `uncaughtException`. A corrupted process can be worse than a controlled restart. Let the runtime/deployment platform restore the service.

---

## 9. Database and repository boundaries

Application code should not depend on ORM details unless the project deliberately chooses that architecture.

Prefer:

```ts
export abstract class UsersRepository {
  abstract findById(id: string): Promise<User | null>;
  abstract findByEmail(email: string): Promise<User | null>;
  abstract create(input: CreateUserRepositoryInput): Promise<User>;
}
```

Then provide an infrastructure implementation:

```ts
@Injectable()
export class TypeOrmUsersRepository implements UsersRepository {
  // TypeORM-specific implementation
}
```

Do not introduce repository abstractions mechanically for simple CRUD modules. Apply the abstraction when it protects application logic from infrastructure, improves testing, or creates a real substitution boundary.

Repository rules:

- no HTTP response objects
- no controller dependencies
- no business decisions that belong to the application/domain layer
- explicit transaction boundaries for operations that must be atomic
- avoid N+1 queries
- select only what is needed for sensitive or expensive data where appropriate

---

## 10. Logging and observability

Use Nest's `Logger` or the project's structured logger consistently.

Good log entry:

```ts
this.logger.error('Failed to synchronize marketplace order', {
  orderId,
  marketplace,
  cause: error instanceof Error ? error.message : 'unknown',
});
```

Avoid:

```ts
console.log(JSON.stringify(entireRequest));
```

Never log passwords, authorization headers, tokens, card data, secrets, or entire sensitive payloads.

At application boundaries, preserve enough context to correlate failures with a request/job/message. Prefer an existing request ID or trace ID from the application's observability stack instead of inventing a second correlation system.

---

## 11. Naming and file organization

Use explicit names:

```text
create-user.service.ts
users.repository.ts
typeorm-users.repository.ts
all-exceptions.filter.ts
create-user.dto.ts
user.response.ts
```

Prefer verb-based names for use cases:

- `CreateUserService`
- `AuthenticateUserService`
- `SyncMarketplaceOrdersService`
- `CancelOrderService`

Avoid generic names when behavior is specific:

- `Manager`
- `Helper`
- `Utils`
- `CommonService`
- `DataService`

A utility is acceptable when it truly represents a stateless reusable operation and its abstraction is stable.

---

## 12. Dependency injection rules

Prefer constructor injection:

```ts
@Injectable()
export class OrdersService {
  constructor(
    private readonly ordersRepository: OrdersRepository,
    private readonly paymentGateway: PaymentGateway,
  ) {}
}
```

Avoid service locator patterns such as retrieving arbitrary providers from `ModuleRef` unless the framework's dynamic behavior actually requires it.

Avoid circular dependencies as an architectural smell. Before using `forwardRef()`, check whether responsibilities can be moved into a third abstraction or the module boundary can be redesigned.

---

## 13. External integrations

Treat third-party systems as unreliable boundaries.

For HTTP APIs, queues, marketplaces, payment gateways, FTP/SFTP, cloud APIs, etc.:

- set explicit timeouts
- validate external responses
- translate provider-specific errors into stable application errors
- avoid leaking provider details to API clients
- use retries only when the operation is safe to retry
- use idempotency for operations that can be delivered more than once
- use exponential backoff where appropriate
- make pagination/batching explicit
- handle connection resets and partial failures
- log enough context to investigate the failure

Never assume a successful TCP/HTTP response means the business operation succeeded.

---

## 14. Configuration and secrets

- Read configuration from environment/config providers.
- Validate required configuration at startup.
- Never commit secrets.
- Never hard-code credentials.
- Keep configuration access centralized.
- Do not scatter `process.env.X` throughout business logic.

Prefer:

```ts
const apiUrl = this.configService.getOrThrow<string>('MARKETPLACE_API_URL');
```

over direct environment access throughout services.

---

## 15. Testing

### Unit tests

Test behavior, not implementation details.

Good test:

```ts
it('throws ConflictException when the email already exists', async () => {
  repository.findByEmail.mockResolvedValue(existingUser);

  await expect(service.execute(input)).rejects.toBeInstanceOf(
    ConflictException,
  );
});
```

Avoid asserting internal private methods or exact call counts unless those calls are part of the contract being tested.

### Required exception-filter tests

The global filter should have tests covering at least:

1. known `HttpException` preserves its status
2. unknown `Error` becomes HTTP 500
3. unknown non-Error value becomes HTTP 500
4. response body does not expose stack traces
5. logging occurs for unexpected errors
6. safe error details remain intact when using the project's standard error schema

### Integration/e2e tests

Cover transport boundaries:

- validation
- authentication/authorization
- HTTP status codes
- response schema
- database integration where appropriate
- external integration contracts where practical

---

## 16. Refactoring checklist

Before finishing a refactor, verify:

```text
[ ] No `any` introduced
[ ] Controllers remain thin
[ ] Business logic is outside controllers
[ ] No unnecessary try/catch blocks
[ ] Exceptions use meaningful HTTP semantics
[ ] Global exception filter exists and is registered
[ ] Unexpected errors return a safe 500 response
[ ] Secrets/internal details are not exposed
[ ] External integrations have explicit failure behavior
[ ] Dependencies are injected
[ ] No circular dependency was introduced
[ ] DTO/input validation is enforced
[ ] Tests cover changed behavior
[ ] No unrelated refactor was mixed into the change
```

---

## 17. Code-review behavior for an AI coding agent

When reviewing or generating NestJS code:

1. First understand the existing architecture and conventions.
2. Preserve established patterns when they are sound.
3. Identify the smallest clean change that solves the task.
4. Reject unnecessary abstractions.
5. Flag `any`, unsafe casts, duplicated business rules, fat controllers, hidden side effects, swallowed errors, and unbounded retries.
6. Check error behavior explicitly, including unexpected exceptions.
7. Check whether the new code introduces an architectural dependency in the wrong direction.
8. Add or update tests for meaningful behavior.
9. Run the narrowest relevant test suite first, then broader checks when practical.
10. Do not declare success without verifying the code compiles/tests pass when execution is available.

---

## 18. Definition of Done

A NestJS change is complete when:

- the code is idiomatic TypeScript/NestJS
- responsibilities are appropriately separated
- SOLID principles are applied where they add value
- external input is validated
- expected failures use intentional exceptions
- unexpected request-level failures are caught by the global filter
- the API never leaks internal exception details
- process-level resilience is handled by lifecycle/runtime infrastructure rather than pretending an exception filter can prevent every crash
- tests cover the changed behavior
- lint/typecheck/tests pass when those checks are available
- no unrelated behavior was changed

---

## Official references

Use the current NestJS documentation as the authoritative framework reference, especially for exception filters, validation, providers, modules, pipes, guards, interceptors, testing, and lifecycle behavior.

- https://docs.nestjs.com/exception-filters
- https://docs.nestjs.com/application/validation
- https://docs.nestjs.com/providers
- https://docs.nestjs.com/techniques
