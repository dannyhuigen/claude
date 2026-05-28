# Coding Rules (Slim)

Slim version of `CLAUDE.md` — rules only, no rationale.

## General

### Do
- Keep variables, constants, and methods private (lowercase) whenever possible.
- Split a file or component once it exceeds ~500 LOC or ~10 functions.
- Follow Effective Go and Google Go Style as the baseline for code style.
- Write all prose and identifiers in British (Oxford) English.
- Use meaningful, specific names without unnecessary filler words.
- Name methods after what they do, not how they do it.
- Use the full constructor name `New<Type>` regardless of how many types the package exports.
- Define interfaces where they are implemented, not where consumed.
- Use single-letter receivers (except `ac` for `AdminController`).
- Spell out names in full except for the agreed abbreviations (`err`, `fn`, `ctx`, `tx`, `conn`, `ctrl`, `svc`, `repo`, `db`, `deps`).
- Prefix enumerated constants with their type name (e.g. `LogLevelError`).
- Use kebab-case for stringified enum values.
- Use singular names for packages, folders, and files except where only plural applies (e.g. `routes.go`).
- Use camelCase/PascalCase for Go variable and function names.
- Convert times to UTC as early as possible and always call `.UTC()` before passing `time.Time` to GORM.
- Return value types for data structs like models and DTOs by default.
- Store configuration and static data in code by default.

### Don't
- Don't use global variables.
- Don't use the `ginCtx` name or similar for context.
- Don't omit types in declarations even when repeated (e.g. `arg1, arg2 string`).
- Don't use `goto` statements, though labels to break nested loops are OK.
- Don't return pointers for data structs unless the struct is large, absence must be represented, or the caller must mutate it.
- Don't store data in the database unless it must differ per environment, change without a deploy, or be edited by a non-technical person.

## Architecture and Layers

### Do
- Separate code first by layer (controller/service/repository) then by sub-feature.
- Keep layers loosely coupled.
- Omit the feature prefix on dependencies of the same feature (e.g. `service` not `verificationService` inside `verification`).

### Don't
- Don't reference HTTP-related types in the repository or service layers.
- Don't reference database-specific types in the service layer.

## Components

### Do
- Build every component as a public interface, a private struct of dependencies, and a public `New<Component>` constructor.
- Wire all components together through dependency injection.
- Name a component after what it does (`Client`, `CRON`, `Middleware`, `DateOrganizer`, etc.), not forced into "service".
- Create a new component for a new domain concept, an oversized existing component, or a new external integration.

## Models

### Do
- Keep base domain models to their own scalar fields and reference relations by ID only, except 1-to-1/1-to-0 relationships.
- Put always-needed related entities in the default model and rarely-needed data in extended models.
- Use standard Go types (`time.Time`, `*string`, `enums.*`, etc.) in models.
- Nest the base version of a related entity.
- Build extended domain models only in the service layer.

### Don't
- Don't define business-logic methods on models.
- Don't put `json`, `gorm`, or other layer-specific tags on domain models.

## Repositories

### Do
- Give every repository sub-feature a `repository.go` (interface, struct, constructor), a `structs.go` (row structs), and a `transform.go` (row↔model conversion).
- Embed `models.Base` and define `TableName()` and `ModelName()` on row structs.
- Validate in `rowToModel` that required preloads are present and return an error if not.
- Return domain models only from the repository interface.
- Access data from another feature through that feature's service, not its repository.
- Batch queries with a `batch_size` well below the 65K parameter limit.
- Set up join tables manually with `SetupJoinTable` when soft-deletion must be respected.
- Guard `Create(slice)` calls against empty slices.
- Use `Select("*")` or name the field when you must force-update a zero-value field.
- Match `select_field` tags to the CamelCase variant of the column name.
- Build `ORDER BY` and `WHERE` clauses with nested expressions as plain strings (inlining Go constants via `fmt.Sprintf`).
- Export embedded structs and their fields when updating their definitions.

### Don't
- Don't leak row structs outside the repository package, except for auto-migrate.
- Don't use GORM outside the repository layer, except in migrations.
- Don't preload more than 2 levels deep, and only preload 1-to-1/1-to-0 base relationships at the second level.
- Don't reuse a Go field name that collides with an embedded-struct field under `embeddedPrefix`.

## Services

### Do
- Give every service sub-feature a `service.go` defining the interface, struct, constructor, and methods.
- Depend on other features' services rather than their repositories.

### Don't
- Don't access the database directly in the service layer.

## Controllers

### Do
- Give every controller sub-feature a `controller.go` (constructor, public `Controller` struct, methods, DTOs) and a `routes.go`.
- Define separate `AdminController` and `PartnerController` types in their own files when those surfaces exist.
- Use `SendResponse()` for all API responses.
- Register a controller by adding it to `di/controller.go`, the `ControllerContainer`, and `cmd/api/routes.go`.
- Follow REST/CRUD with plural resource nouns and HTTP methods for actions.
- Use an explicit verb suffix only when the operation exceeds simple CRUD (e.g. `PUT /group-dates/join-waitlist`).
- Use `POST /resources/:id/search` for complex filter parameters.
- Convert models to view structs in `dto/responses/` before returning JSON, using a presenter when conversion involves business logic.
- Include the updated model in the response of any mutation endpoint.
- Add `omitempty` to every `string` view-model field.
- Initialise response slices with `make([]T, 0)`.

### Don't
- Don't end an endpoint path with a trailing `/`.
- Don't use verbs in URLs when an HTTP method expresses the action.
- Don't return domain models directly as JSON.

## Sensitive User Data in Admin Endpoints

### Do
- Return sensitive user data only from `GET /admin/users/:id/complete`.
- Convert any `*models.User` reference with `responses.NewEmbeddedUser(user)`.
- Have the client call `/complete` separately when a feature needs sensitive data.
- Prefer a reveal-button pattern over auto-loading when a new page displays user data.

### Don't
- Don't use a `models.*` type as a field in any admin response DTO (converter signatures excepted).
- Don't add sensitive user data (phone, email, birthday, gender, address, medias, bio, Q&A, work, education, height, sexuality, preferences, tags) to any admin response DTO.
- Don't return raw models from admin endpoints.

## Errors

### Do
- Use `breeze-api/util/errors` for all error handling.
- Check the error return value immediately after each fallible call.
- Wrap errors crossing a component boundary (including third-party packages like GORM) to add context.
- Either fully handle an error or propagate it, never both.
- Make ignored errors explicit with `_ =` or a commented justification.
- Prefix string-valued sentinel errors with `Err`/`err` and suffix error types with `Error`.
- Compare errors with `errors.Is`/`errors.As`.
- Return `errors.NotFound` (e.g. `errors.NewNotFound("user", id)`) when a resource is missing.

### Don't
- Don't wrap errors from methods within the same service or repository.
- Don't use string formatting to inject dynamic values into an error message.
- Don't return `nil, nil` or pointer returns to signal "not found".
- Don't return `gorm.ErrRecordNotFound` directly or use it outside the repository layer.

## Logging

### Do
- Pass loggers via context.
- Add newly available useful data to the logger and restore it in the context.
- Log errors with `errors.LogError(err, logging.FromContext(ctx))`.

## Comments

### Do
- Add a comment when it is unclear why or what is happening, or when naming is not self-explanatory.
- Format TODOs as `// TODO(@yourGitHubAlias): ...` with a corresponding ClickUp ticket.

### Don't
- Don't comment code that is small and easy to read.

## Enums and Feature Flags

### Do
- Run `make enums` after adding or removing enums in `enums/enums.go`.
- Add a post-migration using `softDeleteTextTemplate` when removing a `TextTemplateName`.
- Create new feature flags as `FeatureFlagName` accessed through the feature flag service.
- Name feature flags in kebab-case following `<action>_<feature>` (e.g. `show_payment_usps`).
- Remove a feature flag (definition, code paths, then `make enums`) once its feature has been stable for 2+ weeks.

### Don't
- Don't keep feature flags for completed migrations, fully rolled-out features, or concluded A/B tests.

## Migrations

### Do
- Place data migrations in `/migrations/pre/` (before schema changes) or `/migrations/post/` (after), defaulting to post.
- Export each migration file's function returning `map[string]func() error` and register it in `migrations/migration.go`.
- Define constraints and indexes declaratively via GORM tags, `constraint.go`, or a data-cleanup pre-migration.
- Use a pre-migration with `DROP COLUMN IF EXISTS` to drop columns.
- Keep database migrations backwards compatible across a rolling deploy.

### Don't
- Don't `CREATE COLUMN` or `ALTER COLUMN ... TYPE` in a migration.
- Don't add constraints or indexes via one-off migrations.
- Don't drop a database column in the same release that removes its code usage.

## Database

### Do
- Add `ON DELETE CASCADE` to every foreign key.
- Prefer singular column indexes over composite ones.
- Use database triggers to update the `update_flags` table when related rows change.

## External Integrations

### Do
- Keep the number of external Go dependencies to a minimum.
- Ping #backend-guild before adding a new external dependency.
- Prefer writing a record to the DB and processing it in an async loop over calling an external service synchronously.
- Choose explicit delivery semantics (at-least-once for idempotent calls, at-most-once or idempotency keys otherwise).
- Persist a final status (`Failed`/`Sent`) for every async item and give failed items a recovery path.

### Don't
- Don't call external systems synchronously unless the caller strictly needs an immediate response (e.g. a payment redirect URL).
- Don't silently discard a failed external operation.

## Dependency Injection

### Do
- Use Wire for dependency injection on new code.
- Add a dependency's constructor to the appropriate `di/` file, bind its interfaces to implementations, and add it to a container when it must be accessible.
- Run `make wire` after adding or modifying a dependency provider.

## Testing

### Do
- Place each test in `<component>_test.go` next to the file it tests, using package `<package>_test`.
- Add a `main_test.go` with `TestMain` for required global setup.
- Name test functions `Test_<Component>_<Method>` and table-driven cases with the `when ... should ...` convention.
- Use table-driven tests for multiple inputs, error conditions, and boundaries, following Arrange-Act-Assert.
- Generate test data via the `util/generators` builder package, extending it as needed.
- Use deterministic test data, setting a fixed seed if randomness is unavoidable.
- Use `require` when later assertions would be meaningless on failure and `assert` otherwise.
- Replace dependencies with generated mocks (via `//go:generate mockgen`) and group them in a `dependencies` struct built by a `setup(t)` function.
- Call `setup(t)` per test case for isolation, and use `setupMockDefaults`/`overrideMocks` for happy-path and case-specific behaviour.
- Mock the `TimeProvider` to control time in tests.
- Mark integration tests with `//go:build integration` and wire them via `di.InitializeTest*` initializers.
- Have each integration test run and clean up its own migrations with `testhelper.RunMigrations`/`CleanupMigrations`.
- Make tests parallelisable and independent of execution order.
- Favour more unit tests than integration tests.
- Use custom assertions like `AssertJSONContains`, `AssertEqualSlices`, and `AssertEqualIgnoringBase` where applicable.
- Add `//go:build integration` to a `main_test.go` that only hosts a `TestMain` wrapper for integration tests.

### Don't
- Don't use `time.Sleep` in tests.
- Don't rely on real databases or network calls in unit tests unless that interaction is the thing under test.
- Don't test private methods by making them public.
- Don't over-rely on verifying mock calls or use `Times(n)` on getter calls.
- Don't duplicate implementation logic in tests.
- Don't write tests for trivial getters/setters, generated code (mocks, wire, enums), or third-party internals.

## Git and GitHub

### Do
- Name branches `<type>/<description>(-CU-<clickup-task-id>)` with `type` one of `feat`, `fix`, `chore`.
- Title PRs `<type>: <description> (<clickup-task-id>)` (using `release` for `base: prod`).
- Commit large refactors (renames, moves) separately from logic changes.
- Write meaningful commit messages and PR descriptions explaining what changed and why.
- Resolve or reply to every Claude PR comment before requesting human review.
- Assign yourself to your PR and use only the `awaiting other` and `important` labels.
- Squash-and-merge into `main` and use a merge commit into `prod`.
- Merge `main` into `prod` soon after merging to `main`, releasing risky changes before 3 PM.

### Don't
- Don't add dynamic values via string formatting, trailing slashes, or other contract-breaking patterns to released code without staging validation.
- Don't open a non-draft `main`→`prod` PR when changes are not production-ready.

## Tooling

### Do
- Open the Bruno API client against `127.0.0.1` rather than `localhost`.
- Use Go 1.25 specifically on macOS 26+ if you hit the `missing LC_UUID load command` error.
