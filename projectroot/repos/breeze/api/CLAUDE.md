# Coding Rules

Rules for writing code in the Breeze API repo, synthesised from `docs/`. Each rule is one imperative sentence with a rationale.

## General

### Do
- Keep variables, constants, and methods private (lowercase) whenever possible — because limiting exported surface area reduces coupling and accidental misuse.
- Split a file or component once it exceeds ~500 LOC or ~10 functions — because oversized units are hard to read, test, and maintain.
- Follow Effective Go and Google Go Style as the baseline for code style — because the common Go tooling already adheres to them.
- Write all prose and identifiers in British (Oxford) English — because the codebase standardises on one spelling convention.
- Use meaningful, specific names without unnecessary filler words — because explicit names make intent clear.
- Name methods after what they do, not how they do it — because the "how" is an implementation detail that may change.
- Use the full constructor name `New<Type>` regardless of how many types the package exports — because it keeps construction discoverable and consistent.
- Define interfaces where they are implemented, not where consumed — because it makes mocking easier and avoids duplicated interfaces in search.
- Use single-letter receivers (except `ac` for `AdminController`) — because short receivers are idiomatic and the exception keeps admin controllers distinguishable.
- Spell out names in full except for the agreed abbreviations (`err`, `fn`, `ctx`, `tx`, `conn`, `ctrl`, `svc`, `repo`, `db`, `deps`) — because consistent naming aids readability and search.
- Prefix enumerated constants with their type name (e.g. `LogLevelError`) — because it makes them readable and discoverable.
- Use kebab-case for stringified enum values — because it is the project-wide convention for enum strings.
- Use singular names for packages, folders, and files except where only plural applies (e.g. `routes.go`) — because singular is the project convention.
- Use camelCase/PascalCase for Go variable and function names — because snake_case is reserved for SQL snippets and test names only.
- Convert times to UTC as early as possible and always call `.UTC()` before passing `time.Time` to GORM — because it prevents timezone drift in stored data.
- Return value types for data structs like models and DTOs by default — because copying small structs is cheap and avoids nil-handling overhead.
- Store configuration and static data in code by default — because type-safe enums/structs are easier to work with and avoid environment drift.

### Don't
- Don't use global variables — because they create hidden coupling and make testing harder.
- Don't use the `ginCtx` name or similar for context — because `ctx` is the standard name.
- Don't omit types in declarations even when repeated (e.g. `arg1, arg2 string`) — because explicit types improve clarity.
- Don't use `goto` statements, though labels to break nested loops are OK — because `goto` harms control-flow readability.
- Don't return pointers for data structs unless the struct is large, absence must be represented, or the caller must mutate it — because needless pointers add nil-handling complexity.
- Don't store data in the database unless it must differ per environment, change without a deploy, or be edited by a non-technical person — because code is easier to evolve and keeps environments in sync.

## Architecture and Layers

### Do
- Separate code first by layer (controller/service/repository) then by sub-feature — because it makes related code easy to locate and change cohesively.
- Keep layers loosely coupled — because tight coupling between layers makes changes ripple unpredictably.
- Omit the feature prefix on dependencies of the same feature (e.g. `service` not `verificationService` inside `verification`) — because the prefix is redundant within its own package.

### Don't
- Don't reference HTTP-related types in the repository or service layers — because those layers must stay independent of the transport.
- Don't reference database-specific types in the service layer — because persistence details belong in the repository layer.

## Components

### Do
- Build every component as a public interface, a private struct of dependencies, and a public `New<Component>` constructor — because the uniform pattern makes all components wireable and mockable.
- Wire all components together through dependency injection — because it keeps construction centralised and testable.
- Name a component after what it does (`Client`, `CRON`, `Middleware`, `DateOrganizer`, etc.), not forced into "service" — because the name should reflect its role.
- Create a new component for a new domain concept, an oversized existing component, or a new external integration — because these are the cases where a separate unit earns its keep.

## Models

### Do
- Keep base domain models to their own scalar fields and reference relations by ID only, except 1-to-1/1-to-0 relationships — because it avoids deep nesting chains.
- Put always-needed related entities in the default model and rarely-needed data in extended models — because tiering keeps the common case lean.
- Use standard Go types (`time.Time`, `*string`, `enums.*`, etc.) in models — because models must stay layer-agnostic.
- Nest the base version of a related entity — because it avoids deep preload chains like `Match.MatchedUsers[].User.Profile.Medias`.
- Build extended domain models only in the service layer — because the repository layer should not assemble them.

### Don't
- Don't define business-logic methods on models — because that logic belongs in the service layer.
- Don't put `json`, `gorm`, or other layer-specific tags on domain models — because serialisation belongs to view structs.

## Repositories

### Do
- Give every repository sub-feature a `repository.go` (interface, struct, constructor), a `structs.go` (row structs), and a `transform.go` (row↔model conversion) — because the consistent layout makes data access predictable.
- Embed `models.Base` and define `TableName()` and `ModelName()` on row structs — because GORM and the framework rely on them.
- Validate in `rowToModel` that required preloads are present and return an error if not — because missing preloads otherwise produce silently incomplete models.
- Return domain models only from the repository interface — because row structs are an internal representation.
- Access data from another feature through that feature's service, not its repository — because cross-feature repository dependencies create tight coupling.
- Batch queries with a `batch_size` well below the 65K parameter limit — because preloads expand to large `IN` lists that can exceed Postgres limits.
- Set up join tables manually with `SetupJoinTable` when soft-deletion must be respected — because an explicit many2many model does not honour soft-deletes by default.
- Guard `Create(slice)` calls against empty slices — because GORM errors on an empty slice.
- Use `Select("*")` or name the field when you must force-update a zero-value field — because GORM skips zero-value fields by default.
- Match `select_field` tags to the CamelCase variant of the column name — because a mismatch silently yields only `null` values.
- Build `ORDER BY` and `WHERE` clauses with nested expressions as plain strings (inlining Go constants via `fmt.Sprintf`) — because nested `gorm.Expr` is silently dropped on GORM v1.25.
- Export embedded structs and their fields when updating their definitions — because GORM automigrate ignores unexported embedded structs.

### Don't
- Don't leak row structs outside the repository package, except for auto-migrate — because they are an internal database representation.
- Don't use GORM outside the repository layer, except in migrations — because the ORM must stay confined to persistence.
- Don't preload more than 2 levels deep, and only preload 1-to-1/1-to-0 base relationships at the second level — because deeper preloads belong in service-layer composition.
- Don't reuse a Go field name that collides with an embedded-struct field under `embeddedPrefix` — because the embedded entry overwrites the parent in GORM's schema map and breaks `Select`/`autoCreateTime`.

## Services

### Do
- Give every service sub-feature a `service.go` defining the interface, struct, constructor, and methods — because the uniform layout keeps business logic discoverable.
- Depend on other features' services rather than their repositories — because it maintains layer separation and prevents tight coupling.

### Don't
- Don't access the database directly in the service layer — because all data operations must go through the repository layer.

## Controllers

### Do
- Give every controller sub-feature a `controller.go` (constructor, public `Controller` struct, methods, DTOs) and a `routes.go` — because Wire and routing rely on this layout.
- Define separate `AdminController` and `PartnerController` types in their own files when those surfaces exist — because each audience needs its own constructor, methods, and DTOs.
- Use `SendResponse()` for all API responses — because it produces consistent response envelopes.
- Register a controller by adding it to `di/controller.go`, the `ControllerContainer`, and `cmd/api/routes.go` — because all three are required for the route to be wired.
- Follow REST/CRUD with plural resource nouns and HTTP methods for actions — because consistent routing makes endpoints predictable.
- Use an explicit verb suffix only when the operation exceeds simple CRUD (e.g. `PUT /group-dates/join-waitlist`) — because verbs are clearer than nouns for side-effecting domain actions.
- Use `POST /resources/:id/search` for complex filter parameters — because GET cannot carry complex filter bodies.
- Convert models to view structs in `dto/responses/` before returning JSON, using a presenter when conversion involves business logic — because view structs decouple the API contract from domain models.
- Include the updated model in the response of any mutation endpoint — because clients need the post-mutation state.
- Add `omitempty` to every `string` view-model field — because an empty string should never represent absence.
- Initialise response slices with `make([]T, 0)` — because it serialises as `[]` rather than `null`.

### Don't
- Don't end an endpoint path with a trailing `/` — because it breaks routing consistency.
- Don't use verbs in URLs when an HTTP method expresses the action — because REST nouns are the default.
- Don't return domain models directly as JSON — because view structs must mediate the API contract.

## Sensitive User Data in Admin Endpoints

### Do
- Return sensitive user data only from `GET /admin/users/:id/complete` — because centralising it allows rate-limiting, auditing, and monitoring in one place.
- Convert any `*models.User` reference with `responses.NewEmbeddedUser(user)` — because it produces a safe, non-sensitive reference.
- Have the client call `/complete` separately when a feature needs sensitive data — because no other endpoint may expose it.
- Prefer a reveal-button pattern over auto-loading when a new page displays user data — because sensitive data should be fetched only on explicit request.

### Don't
- Don't use a `models.*` type as a field in any admin response DTO (converter signatures excepted) — because a future User preload on that model would leak sensitive data into JSON.
- Don't add sensitive user data (phone, email, birthday, gender, address, medias, bio, Q&A, work, education, height, sexuality, preferences, tags) to any admin response DTO — because only `/complete` may return it.
- Don't return raw models from admin endpoints — because every response must pass through a `dto/responses/` DTO.

## Errors

### Do
- Use `breeze-api/util/errors` for all error handling — because the standard `errors` package lacks stack traces and Sentry context.
- Check the error return value immediately after each fallible call — because it keeps the happy path separate from error handling.
- Wrap errors crossing a component boundary (including third-party packages like GORM) to add context — because wrapping adds description, stack traces, and searchable Sentry data.
- Either fully handle an error or propagate it, never both — because mixing the two leads to double-handling and confusion.
- Make ignored errors explicit with `_ =` or a commented justification — because silent swallowing hides failures.
- Prefix string-valued sentinel errors with `Err`/`err` and suffix error types with `Error` — because consistent naming aids recognition.
- Compare errors with `errors.Is`/`errors.As` — because direct `==` comparison is unsafe against wrapped errors.
- Return `errors.NotFound` (e.g. `errors.NewNotFound("user", id)`) when a resource is missing — because it signals absence explicitly without ambiguous nil returns.

### Don't
- Don't wrap errors from methods within the same service or repository — because the boundary is where context should be added.
- Don't use string formatting to inject dynamic values into an error message — because it breaks Sentry error bucketing; use tags and fields instead.
- Don't return `nil, nil` or pointer returns to signal "not found" — because callers then must check two values ambiguously.
- Don't return `gorm.ErrRecordNotFound` directly or use it outside the repository layer — because persistence errors must not leak upward.

## Logging

### Do
- Pass loggers via context — because it propagates request-scoped logging metadata.
- Add newly available useful data to the logger and restore it in the context — because downstream logic then logs richer information.
- Log errors with `errors.LogError(err, logging.FromContext(ctx))` — because it includes attached tags/fields and skips noise like `context canceled`.

## Comments

### Do
- Add a comment when it is unclear why or what is happening, or when naming is not self-explanatory — because comments should cover what the code cannot.
- Format TODOs as `// TODO(@yourGitHubAlias): ...` with a corresponding ClickUp ticket — because untracked TODOs are never resolved.

### Don't
- Don't comment code that is small and easy to read — because redundant comments add noise.

## Enums and Feature Flags

### Do
- Run `make enums` after adding or removing enums in `enums/enums.go` — because the validators in `validators_generated.go` are generated.
- Add a post-migration using `softDeleteTextTemplate` when removing a `TextTemplateName` — because seeded rows, translations, and AB versions otherwise remain in the database.
- Create new feature flags as `FeatureFlagName` accessed through the feature flag service — because the project is migrating away from settings-based flags.
- Name feature flags in kebab-case following `<action>_<feature>` (e.g. `show_payment_usps`) — because it keeps flags descriptive and consistent.
- Remove a feature flag (definition, code paths, then `make enums`) once its feature has been stable for 2+ weeks — because stale flags accumulate dead code.

### Don't
- Don't keep feature flags for completed migrations, fully rolled-out features, or concluded A/B tests — because only kill switches and toggleable integrations justify permanence.

## Migrations

### Do
- Place data migrations in `/migrations/pre/` (before schema changes) or `/migrations/post/` (after), defaulting to post — because pre-migrations should only be used when schema timing requires it.
- Export each migration file's function returning `map[string]func() error` and register it in `migrations/migration.go` — because unregistered migrations never run.
- Define constraints and indexes declaratively via GORM tags, `constraint.go`, or a data-cleanup pre-migration — because they must be re-created on every boot rather than run once.
- Use a pre-migration with `DROP COLUMN IF EXISTS` to drop columns — because a post-migration drop runs after auto-migrate and GORM re-adds the column.
- Keep database migrations backwards compatible across a rolling deploy — because old and new instances run side by side.

### Don't
- Don't `CREATE COLUMN` or `ALTER COLUMN ... TYPE` in a migration — because column existence and type are owned by GORM and a mismatch triggers an `ALTER` on every boot.
- Don't add constraints or indexes via one-off migrations — because those run once per environment and won't exist on fresh databases.
- Don't drop a database column in the same release that removes its code usage — because still-running old instances expect the column.

## Database

### Do
- Add `ON DELETE CASCADE` to every foreign key — because orphaned rows must be cleaned up automatically.
- Prefer singular column indexes over composite ones — because the performance difference is negligible and singular indexes are more flexible.
- Use database triggers to update the `update_flags` table when related rows change — because it is the established mechanism for change tracking.

## External Integrations

### Do
- Keep the number of external Go dependencies to a minimum — because each dependency adds maintenance and supply-chain risk.
- Ping #backend-guild before adding a new external dependency — because additions need team feedback.
- Prefer writing a record to the DB and processing it in an async loop over calling an external service synchronously — because it keeps the request path fast and decouples it from the external call.
- Choose explicit delivery semantics (at-least-once for idempotent calls, at-most-once or idempotency keys otherwise) — because DB writes and external calls can never be atomic.
- Persist a final status (`Failed`/`Sent`) for every async item and give failed items a recovery path — because no failure should be silently lost.

### Don't
- Don't call external systems synchronously unless the caller strictly needs an immediate response (e.g. a payment redirect URL) — because synchronous external calls slow and couple the request path.
- Don't silently discard a failed external operation — because failed items must remain visible and recoverable.

## Dependency Injection

### Do
- Use Wire for dependency injection on new code — because the project is standardising on it.
- Add a dependency's constructor to the appropriate `di/` file, bind its interfaces to implementations, and add it to a container when it must be accessible — because Wire needs all three to provide it.
- Run `make wire` after adding or modifying a dependency provider — because the generated `wire_gen.go` files must be regenerated.

## Testing

### Do
- Place each test in `<component>_test.go` next to the file it tests, using package `<package>_test` — because external test packages keep tests against the public interface.
- Add a `main_test.go` with `TestMain` for required global setup — because the framework runs it before the package's tests.
- Name test functions `Test_<Component>_<Method>` and table-driven cases with the `when ... should ...` convention — because descriptive names make failures self-explanatory.
- Use table-driven tests for multiple inputs, error conditions, and boundaries, following Arrange-Act-Assert — because the structure keeps cases concise and comparable.
- Generate test data via the `util/generators` builder package, extending it as needed — because shared builders keep defaults consistent and tests minimal.
- Use deterministic test data, setting a fixed seed if randomness is unavoidable — because tests must produce consistent results across runs.
- Use `require` when later assertions would be meaningless on failure and `assert` otherwise — because it avoids panics from continuing past a fatal failure.
- Replace dependencies with generated mocks (via `//go:generate mockgen`) and group them in a `dependencies` struct built by a `setup(t)` function — because it isolates the unit under test.
- Call `setup(t)` per test case for isolation, and use `setupMockDefaults`/`overrideMocks` for happy-path and case-specific behaviour — because per-case setup prevents cross-case leakage.
- Mock the `TimeProvider` to control time in tests — because it removes reliance on real wall-clock time.
- Mark integration tests with `//go:build integration` and wire them via `di.InitializeTest*` initializers — because the build tag separates them and the initializers stay in sync with production DI.
- Have each integration test run and clean up its own migrations with `testhelper.RunMigrations`/`CleanupMigrations` — because tests must not depend on shared leftover state.
- Make tests parallelisable and independent of execution order — because order-dependent tests are flaky.
- Favour more unit tests than integration tests — because the testing pyramid keeps the suite fast.
- Use custom assertions like `AssertJSONContains`, `AssertEqualSlices`, and `AssertEqualIgnoringBase` where applicable — because they encode project-specific comparison semantics.
- Add `//go:build integration` to a `main_test.go` that only hosts a `TestMain` wrapper for integration tests — because otherwise Go builds and runs an empty unit-test binary, wasting link time.

### Don't
- Don't use `time.Sleep` in tests — because it makes tests slow and flaky; mock time instead.
- Don't rely on real databases or network calls in unit tests unless that interaction is the thing under test — because external systems slow tests and add flakiness.
- Don't test private methods by making them public — because you can't verify the public interface uses them correctly.
- Don't over-rely on verifying mock calls or use `Times(n)` on getter calls — because asserting implementation details makes tests brittle.
- Don't duplicate implementation logic in tests — because a test that mirrors the code can't catch its bugs.
- Don't write tests for trivial getters/setters, generated code (mocks, wire, enums), or third-party internals — because they add maintenance without value.

## Git and GitHub

### Do
- Name branches `<type>/<description>(-CU-<clickup-task-id>)` with `type` one of `feat`, `fix`, `chore` — because the Conventional Branch format groups and identifies branches.
- Title PRs `<type>: <description> (<clickup-task-id>)` (using `release` for `base: prod`) — because it mirrors the branch and stays scannable.
- Commit large refactors (renames, moves) separately from logic changes — because it makes reviewing actual logic changes easier.
- Write meaningful commit messages and PR descriptions explaining what changed and why — because reviewers rely on them for context.
- Resolve or reply to every Claude PR comment before requesting human review — because Claude auto-reviews and its comments should be addressed first.
- Assign yourself to your PR and use only the `awaiting other` and `important` labels — because other labels are deprecated.
- Squash-and-merge into `main` and use a merge commit into `prod` — because each base has a deploy action expecting that merge style.
- Merge `main` into `prod` soon after merging to `main`, releasing risky changes before 3 PM — because it keeps releases small and leaves time to fix issues.

### Don't
- Don't add dynamic values via string formatting, trailing slashes, or other contract-breaking patterns to released code without staging validation — because `main` is assumed production-ready.
- Don't open a non-draft `main`→`prod` PR when changes are not production-ready — because a draft PR should block the release until validation completes.

## Tooling

### Do
- Open the Bruno API client against `127.0.0.1` rather than `localhost` — because an IPv4/IPv6 resolution issue makes `localhost` slow.
- Use Go 1.25 specifically on macOS 26+ if you hit the `missing LC_UUID load command` error — because newer toolchains trigger it.

## Conflicts

- TODO comment format: `docs/04-development/01-conventions.md` specifies `// TODO(@youGitHubAlias): ...` while `docs/04-development/03-component.md` (via the broader docs) and the conventions doc both also require a corresponding ClickUp ticket; resolved in favour of the conventions doc's format, with the ClickUp-ticket requirement retained. No substantive contradiction.
- Test taxonomy: `docs/05-testing/*` document the current state (unit tests use `testhelper.TestMain`, only an `integration` build tag exists), whereas `docs/04-development/11-test-architecture-proposal.md` proposes a future three-layer taxonomy (`unit`/`component`/`integration`, split `testhelper`, split `models`). Resolved in favour of the current testing docs as the binding rules; the proposal is forward-looking and not yet in force, so its `component` tag and thin-`testhelper` rules were not adopted as must-dos.
