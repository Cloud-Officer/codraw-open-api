# Code Review: codraw/open-api

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
**composer.json (M5):**
- Added missing runtime dependencies to `require`: `symfony/config`, `symfony/console`, `symfony/dependency-injection`, `symfony/http-foundation`, `symfony/http-kernel`, `symfony/property-access`, `symfony/routing` (all `^6.4.0`).
- Moved `codraw/core` (`^0.39`) from `require-dev` to `require` (hard dependency of `RequestBodyValueResolver`).
- Added a `suggest` section: `ext-zip` (install-sandbox command), `codraw/framework-extra-bundle` (Symfony integration — `codraw/dependency-injection` stays dev-only, matching the codraw-messenger/codraw-aws-tool-kit precedent since `DependencyInjection/OpenApiIntegration` is only loaded by the bundle), `doctrine/doctrine-bundle` (Doctrine object construction/reference handling/inheritance extraction are `class_exists`-guarded optional features).
- `composer validate --no-check-publish` passes.
- Open items (not changed): `php: >=8.5` combined with `symfony/*: ^6.4` still deserves a CI check as noted in M5; no declared dependency was removed.

**Code fixes:**

- **H2** — `LoadFromCacheExtractor` and `StoreInCacheExtractor` now sanitize the cache key with `preg_replace('/[^a-zA-Z0-9_.@-]+/', '-', ...)` before building the cache file path, neutralizing directory traversal from `?scope=`/`{version}` while leaving normal keys (`1.0@default`) unchanged.
- **H3** — `ScopeCleanerListener` assigns `$path->{$method} = null` instead of `unset()`, so typed `PathItem` properties are no longer left uninitialized (previously fataled on the next read in the same dump).
- **H4** — `RequestBodyValueResolver::getBodyData()` rejects valid-JSON scalar bodies with a `BadRequestHttpException` instead of passing them to an `array`-typed method (was a 500 `TypeError`), and casts a possibly-null `Content-Type` header to string before `str_starts_with()`.
- **M1** — `RequestQueryParameterFetcherListener` only applies `+ 0` to `number` query parameters when `is_numeric()`; non-numeric input now flows through to validation instead of throwing a `TypeError` (500).
- **M2** — `ResponseApiExceptionListener` constructor now falls back to `[new HttpExceptionToHttpCodeConverter(), new ConfigurableErrorToHttpCodeConverter()]` when no converters are provided, replacing the dead `??=` that left standalone construction returning 500 for every exception.
- **M3** — `DoctrineObjectConstructor::loadObject()` appends the association name with `[...$path, $name]` instead of the array union `$path + [$name]`, fixing nested `doctrineFindByFieldsMap` lookups.
- **M4** — `JmsDoctrineObjectConstructionCompilerPass` guards with `hasDefinition()` before `getDefinition()`, so container compilation no longer crashes when JMS does not register `jms_serializer.doctrine_object_constructor`.
- **L1** — `OpenApi::validate()` uses `enableAttributeMapping()` instead of the deprecated `enableAnnotationMapping()` (drop-in replacement available since Symfony 6.4).
- **L2** — `InstallSandboxCommand` now throws a clear `RuntimeException` when the zip download fails (instead of passing `false` to `dumpFile()`), and skips zip entries without a `/` (undefined-index) or containing `..` (zip-slip). The `tag` option interpolation into the URL was left as-is (validating it could reject currently-working values).
- **L3** — `SecurityRequirement::__get()` returns `$this->data[$name] ?? null`, matching its `?array` return type without a warning.
- **L4** — `QueryParameter::__construct()` normalizes a single `Constraint` to an array, so `BaseConstraintExtractor`'s `array_filter()` calls no longer throw a `TypeError`.

**Not fixed (deliberately):**

- **H1** — gating the exception `detail` block (and raw 500 messages) on `$this->debug` changes the production JSON error response format that consumers/tooling may currently rely on; needs a deliberate decision and a migration note.

**Validation pass (2026-07-20):**

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts` resolves and installs cleanly with the updated `composer.json` (no constraint adjustments needed).
- Test suite: 129 tests, 420 assertions — all fixes above pass; no test-expectation updates were required. The single failure (`InstallSandboxCommandTest::testExecuteZipError`) is pre-existing and environment-specific (the test hardcodes `/tmp/` while macOS `sys_get_temp_dir()` returns `/var/folders/...`); it fails identically without any of the changes above.
- PHPStan (level per `phpstan.dist.neon`): the changes above introduce no new errors and eliminate 4 previously reported ones (the `@phpstan-ignore-next-line` machinery around `RequestQueryParameterFetcherListener:62` and the dead `??=` in `ResponseApiExceptionListener`). The 21 remaining errors are pre-existing (mostly `DependencyInjection/OpenApiIntegration.php` config-builder typing, missing optional `symfony/security-core`, and test-only notices) and were reproduced on the unmodified tree.
- `markdownlint-cli2` passes on all package Markdown files (auto-fix normalized list spacing in this file only).

---

Reviewed: all PHP source under the package root (113 source files, ~7,900 LOC, plus 29 test files, ~3,500 LOC), `composer.json`, and DI integration. Package generates Swagger 2.0 documentation from Symfony routes/attributes/JMS metadata, and also ships runtime request/response machinery (body deserialization, validation, response serialization, JSON error responses).

## Overall assessment

The extraction architecture is genuinely good: a small `ExtractorInterface` contract with priorities, sub-contexts, tagged-iterator wiring, and a clean event-driven "clean/dump" pipeline. The schema model closely mirrors the Swagger 2.0 spec with helpful docblocks and validator attributes. However, the runtime HTTP components have several real problems: the API exception listener leaks exception details (file paths, messages, exception classes) to clients even with debug disabled; the schema cache builds a filesystem path from unsanitized request input (`?scope=` / `{version}`) that is used both for `require` and for file writes; the scope-filtering cleaner `unset()`s typed properties and leaves `PathItem` objects in a state that fatals on the next read; and malformed-but-valid client JSON produces 500s instead of 400s. Dependency declarations in `composer.json` are also substantially incomplete for the code the package ships.

Grade: **C** — a solid design with notable correctness and security issues in the runtime path.

---

## Findings

### High

#### H1. Exception details (file paths, internal messages, exception class names) leaked to clients regardless of debug mode

`EventListener/ResponseApiExceptionListener.php:105` and `:151-159`

`onKernelException()` always appends `$data['detail'] = $this->getExceptionDetail($error)`. `getExceptionDetail()` unconditionally includes the exception class name, message, `code`, absolute `file` path and `line` (recursively for all `previous` exceptions). Only the stack trace is gated by `$this->debug` (`:161-165`). The top-level `message` (`:98`) also echoes `$error->getMessage()` for arbitrary unhandled 500s (which can contain SQL fragments, DSNs, internal paths, etc.). For any request with `json` format, every exception — including unexpected 500s — discloses internals to the client in production. The `detail` block (and the raw message for non-HTTP exceptions) should be emitted only when `$this->debug` is true.

#### **[FIXED]** H2. Schema cache path is built from unsanitized request input and used for both `require` and file writes

`Extraction/Extractor/Caching/LoadFromCacheExtractor.php:41-51`, `Extraction/Extractor/Caching/StoreInCacheExtractor.php:50-66`, `Extraction/ExtractionContext.php:27-30`, `Controller/OpenApiController.php:52-53`

`ExtractionContext::getCacheKey()` returns `api.version . '@' . api.scope`, and both values come straight from the request in `OpenApiController::apiDocAction()` (`$request->query->get('scope')` and the `{version}` route parameter). The cache extractors then do:

```php
$path = $this->cacheDirectory.'/openApi-'.$cacheKey.'.php';
...
$result = require $path;            // LoadFromCacheExtractor:51
...
$configCache->write(...)            // StoreInCacheExtractor:63 (dumpFile creates intermediate dirs)
```

A scope such as `?scope=../../x` traverses out of `kernel.cache_dir`: `ConfigCache::write()`/`dumpFile()` **creates the intermediate directories**, so an attacker who can reach the doc route can cause `.php` files (content `<?php return unserialize('...');`, with the attacker-controlled version string embedded in the payload) to be written at attacker-chosen locations relative to the cache dir, and `LoadFromCacheExtractor` will `require` such paths on later requests (in prod, `ConfigCache::isFresh()` is just `is_file()`). Caching is **enabled by default** (`DependencyInjection/OpenApiIntegration.php:466`). Even without a full RCE chain this is an arbitrary-file-write/local-file-include primitive driven by query input. The cache key must be sanitized (e.g. hash it, or strip to `[A-Za-z0-9_.-]`).

#### **[FIXED]** H3. `ScopeCleanerListener` unsets typed properties, causing a fatal `Error` later in the same dump

`EventListener/ScopeCleanerListener.php:32`

Operations not matching the scope are removed with `unset($path->{$method})`. `PathItem::$get/$post/...` are *typed* properties (`Schema/PathItem.php:14` etc.); `unset()` puts them back into the *uninitialized* state, and the very next read throws `Error: Typed property ... must not be accessed before initialization`. Verified with a minimal reproduction. `PathItem::getOperations()` reads all seven properties directly (`Schema/PathItem.php:82-90`) and is called afterwards by lower-priority `CleanEvent`/`PreDumpRootSchemaEvent` listeners (`TagCleanerListener:30`, `SchemaSorterListener:26`, `ResponseApiExceptionListener::addErrorDefinition:71`) and by `Root::validateDuplicateOperationId()` during `OpenApi::validate()`. Any scoped documentation request that actually filters an operation out therefore 500s. The listener should assign `null` (`$path->{$method} = null`) instead of `unset()`. There is no test for this listener, which is why it slipped through.

#### **[FIXED]** H4. Valid-JSON scalar request bodies cause a `TypeError` (HTTP 500) instead of a 400

`Request/ValueResolver/RequestBodyValueResolver.php:60-75` and `:82`

For `application/json`, `json_decode($request->getContent(), true)` may return a string, int, float, bool, or null for perfectly valid JSON bodies like `"hello"`, `123`, or `true`. Only `\JsonException` is caught (which maps to `[]`); a scalar result is then passed to `assignPropertiesFromAttribute(Request $request, array $propertiesMap, array $requestData)` whose `array` type declaration triggers an uncaught `TypeError` → 500 Internal Server Error. Any client can trigger this on every endpoint using `#[RequestBody]`. The decoded value should be checked with `\is_array()` and rejected with a `BadRequestHttpException` otherwise. Related minor issue on `:55-57`: `$request->headers->get('Content-Type')` can return `null`, which is passed to `str_starts_with()` (deprecated null coercion) before falling through to the 415 branch.

### Medium

#### **[FIXED]** M1. `number` query parameters throw `TypeError` (HTTP 500) on non-numeric input

`EventListener/RequestQueryParameterFetcherListener.php:62`

`$value = $request->query->get($name) + 0;` — in PHP 8, `"abc" + 0` throws `TypeError: Unsupported operand types`. Any client sending a non-numeric value for a `type: 'number'` query parameter gets a 500 instead of a 400/validation error. Use `is_numeric()` first (or cast with `(float)` and let the validation listener report the error).

#### **[FIXED]** M2. `ResponseApiExceptionListener` fallback converter is dead code; standalone construction maps validation errors to 500

`EventListener/ResponseApiExceptionListener.php:26-31`

```php
private iterable $errorToHttpCodeConverters = [],
...
$this->errorToHttpCodeConverters ??= new ConfigurableErrorToHttpCodeConverter();
```

The promoted property defaults to `[]`, which is never `null`, so the `??=` never executes; and if it ever did, it would assign a single (non-iterable) converter that `getStatusCode()`'s `foreach` (`:176`) would crash on. Consequence: constructing the listener without converters (the documented default) makes *every* exception — including `ConstraintViolationListException` and HTTP exceptions — return 500. Should be `if (!$this->errorToHttpCodeConverters) { $this->errorToHttpCodeConverters = [new HttpExceptionToHttpCodeConverter(), new ConfigurableErrorToHttpCodeConverter()]; }` or similar.

#### **[FIXED]** M3. `DoctrineObjectConstructor` builds the association lookup path with array union instead of append

`Serializer/Construction/DoctrineObjectConstructor.php:121`

```php
$this->loadObject(..., $path + [$name])
```

`$path + [$name]` is a union keyed on `0`; whenever `$path` already has an element at index 0 (i.e. any non-root property), the association name is silently discarded and the recursive call receives the parent's path unchanged. The `doctrineFindByFieldsMap` lookup (`:87-90`) for nested associations therefore matches the wrong key. Should be `[...$path, $name]`.

#### **[FIXED]** M4. Compiler pass crashes when the decorated JMS service does not exist

`DependencyInjection/Compiler/JmsDoctrineObjectConstructionCompilerPass.php:13`

`$container->getDefinition('jms_serializer.doctrine_object_constructor')` throws `ServiceNotFoundException` at compile time if JMSSerializerBundle did not register that service (it only does so when Doctrine is present). The pass should guard with `hasDefinition()` before calling `getDefinition()`.

#### **[FIXED]** M5. `composer.json` omits most runtime dependencies actually used by shipped code

`composer.json:6-14`

The `require` section lists only `jms/serializer`, `phpdocumentor/reflection-docblock`, `symfony/event-dispatcher`, `symfony/filesystem`, `symfony/validator`. But non-dev code hard-depends on:

- `symfony/http-foundation` + `symfony/http-kernel` (all `EventListener/*`, `Request/ValueResolver/*`, `Controller/OpenApiController`)
- `symfony/routing` (`OpenApiController`, `Extraction/Extractor/Symfony/*`, `Versioning/*`)
- `symfony/console` (`Command/InstallSandboxCommand`)
- `symfony/config` (`ConfigCache` in the caching extractors)
- `symfony/property-access` (`RequestBodyValueResolver:15,89`)
- `codraw/core` (`Draw\Component\Core\DynamicArrayObject` in `RequestBodyValueResolver:5,88`) — currently only in `require-dev`
- `symfony/dependency-injection` + `codraw/dependency-injection` (`DependencyInjection/OpenApiIntegration`)

Installed standalone, large parts of the package fatal with class-not-found. Either move these to `require`, or declare them in `suggest` and document the optional feature boundaries. Also note `php: >=8.5` is an unbounded constraint combined with `symfony/*: ^6.4` — Symfony 6.4 predates PHP 8.5, so this combination deserves a CI check or a raised Symfony floor.

### Low

#### **[FIXED]** L1. Deprecated validator API and per-call validator construction

`OpenApi.php:114-118`

`Validation::createValidatorBuilder()->enableAnnotationMapping()` — `enableAnnotationMapping()` is deprecated as of Symfony 6.4 (removed in 7.0); use `enableAttributeMapping()`. Also, a new validator is built on every `validate()` call; harmless for a doc-generation path but wasteful.

#### **[FIXED]** L2. `InstallSandboxCommand` lacks error handling and trusts archive contents

`Command/InstallSandboxCommand.php:48`, `:69-84`

- `file_get_contents('https://github.com/...'.$tag.'.zip')` result is not checked; on network failure it returns `false` and `dumpFile()` receives `false` (writes empty/errors).
- The `tag` option is interpolated into the URL without validation (a crafted value with `/../` can point at a different GitHub archive).
- Extraction concatenates zip entry names into the output path (`:82`); entries containing `../` escape the target directory (zip-slip), and `$explodedPath[1]` (`:74`) is an undefined-index for entries without `/`. Low severity because this is a dev-time command pulling from a trusted org, but sanitizing entry names and checking the download result is cheap.

#### **[FIXED]** L3. `SecurityRequirement::__get()` on a missing key raises a warning

`Schema/SecurityRequirement.php:36`

`return $this->data[$name];` should be `return $this->data[$name] ?? null;` to match its `?array` return type.

#### **[FIXED]** L4. `BaseConstraintExtractor` crashes when `QueryParameter::$constraints` is a single Constraint

`Extraction/Extractor/Constraint/BaseConstraintExtractor.php:47-50`, `:148-151`

`QueryParameter::$constraints` is typed `array|Constraint` (`Schema/QueryParameter.php:16`), but `array_filter($target->constraints, ...)` requires an array; passing `constraints: new NotBlank()` (allowed by the attribute signature) throws a `TypeError` at extraction time. Normalize to an array in the attribute constructor.

---

## Strengths

- **Clean, extensible extraction architecture**: the `ExtractorInterface` + `ExtractionContext` design with priorities, sub-contexts, and `ExtractionCompletedException` short-circuiting is simple and composable; DI autoconfiguration (`ExtractorInterface`, `TypeToSchemaHandlerInterface`, `ReferenceCleanerInterface` tags) makes extension points obvious.
- **Faithful, well-documented schema model**: `Schema/*` classes mirror the Swagger 2.0 spec with spec-quoting docblocks, JMS serialization metadata, and validator attributes including a `GroupSequenceProvider` on `Schema` (`type` vs `$ref` mutual exclusion) and a `Root` callback catching duplicate `operationId`s.
- **Modern PHP attribute-driven API** (`#[RequestBody]`, `#[QueryParameter]`, `#[Serialization]`, `#[Vendor]`, `#[Tag]`) with sensible fallbacks to reflection types and phpDoc.
- **Thoughtful cleaning pipeline**: unreferenced-definition removal iterating to a fixed point, duplicate-definition deduplication, alias renaming, Doctrine inheritance cleanup — all as ordered event listeners that are individually testable.
- **Round-trip fixture testing** in `Tests/OpenApiTest.php` (extract JSON → dump → JSON equality) is a strong invariant test for the serializer layer.
- **Near-empty phpstan baseline** (2 lines) indicates the static-analysis debt has been paid down.

---

## Test Coverage

Roughly 29 test files (~3,500 LOC) against 113 source files (~7,900 LOC). Coverage is decent on the pure/transform layer and weak on the runtime/IO layer:

**Covered**: `OpenApi` core (extract/dump round-trip, validation errors, extractor short-circuit); definition alias and duplicate-definition cleaner listeners; un-reference cleaner; query-parameter fetcher listener; request-validation listener; response serializer listener; response API exception listener; both error-to-HTTP-code converters; PhpDoc `OperationExtractor`; `TypeSchemaExtractor`; JMS `PropertiesExtractor`; `AliasesClassNamingFilter`; version matcher; DI integration config; `InstallSandboxCommand`; `OpenApiController`.

**Not covered (notable gaps)**:

- `RequestBodyValueResolver` — the single most security/robustness-sensitive runtime class has no test (finding H4 would have surfaced).
- `ScopeCleanerListener` — untested; finding H3 is a crash in its main path.
- Caching extractors (`LoadFromCacheExtractor`, `StoreInCacheExtractor`, `FileTrackingExtractor`) — untested, including the cache-key handling of finding H2.
- `DoctrineObjectConstructor` / `ObjectReferenceHandler` / `SimpleObjectConstructor` (entity lookup semantics, finding M3).
- JMS subscribers (`OpenApiSubscriber`, `BasicUnionDeserializerSubscriber`, `GenericSerializerHandler`).
- Constraint extractors other than `NotBlank` (Choice/Range/Length/Count/NotNull).
- `SymfonySchemaBuilder`, `RouterRootSchemaExtractor`/`RouteOperationExtractor`, `InheritanceExtractor`, `SerializationControllerListener`, `SchemaSorterListener`, `JmsDoctrineObjectConstructionCompilerPass`.

Recommendation: prioritize tests for the request-handling path (body resolver with malformed bodies/content types, scoped dump end-to-end, cache key sanitization) since that is where the user-facing bugs cluster.
