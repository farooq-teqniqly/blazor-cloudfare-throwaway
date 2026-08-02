# .NET baseline conventions

Shared conventions for all farooq-teqniqly .NET projects. Source of truth:
[farooq-teqniqly/claude-templates](https://github.com/farooq-teqniqly/claude-templates)
(`CLAUDE-baseline-dotnet.md`). Repos consume a copy of this file imported from their
`CLAUDE.md`; update the template first, then re-sync copies. Project-specific rules
belong in the repo's own `CLAUDE.md`, not here.

## Language and spelling

- U.S. English in all prose, code comments, commit messages, PR descriptions, and docs
  ("color", "behavior", "organize", "canceled", "license" as a noun).
- Do not "correct" .NET API names or existing identifiers that use other spellings
  (e.g. `CancellationToken` stays as-is) -- match identifiers exactly.
- No emojis in docs.
- Prefer hyphens or `--` over em dashes in new prose. Do not sweep existing docs to
  retrofit this preference.
- Never use em dashes or smart quotes inside PowerShell 5.1 string literals -- 5.1 reads
  UTF-8-without-BOM as Windows-1252, so they corrupt and break parsing. This is a hard
  rule, not a style preference.
- In any parameterized PowerShell script, `param()` must be the **very first statement** --
  a comment-based help block may precede it, but no executable statement may.

## Stack defaults

- Target **net10.0**; use the latest C# features where they improve clarity. SDK version
  pinned in `global.json` (`rollForward: latestFeature`).
- Database: each repo names its own store in its `CLAUDE.md`. The default is **SQL Server**
  run locally in **Podman/Docker**, exercised in tests via **Testcontainers**. An embedded
  store (SQLite, LiteDB) is a legitimate choice for a single-user or xcopy-deployed app --
  state the choice and the reason in the repo's `CLAUDE.md` rather than treating it as a
  deviation.
- Tests: **xUnit v3** + **NSubstitute**. Integration and end-to-end tests exercise the real
  store -- a container for a server database, the real file for an embedded one -- **never
  in-memory database fakes** (`UseInMemoryDatabase` and friends), which have different
  semantics from the thing they stand in for.
- **Central Package Management:** all package versions live in `Directory.Packages.props`;
  `.csproj` files reference packages without a version.
- `Directory.Build.props`: `Nullable` + `ImplicitUsings` enabled, `TreatWarningsAsErrors=true`
  (`CodeAnalysisTreatWarningsAsErrors=false` so `CA*` findings warn without failing the build),
  SonarAnalyzer.CSharp everywhere, test packages auto-injected into projects named `*Tests`.
- **Formatting is owned by CSharpier** (local tool in `.config/dotnet-tools.json`, the
  location `dotnet tool restore` discovers by default; use the
  `format`/`check` subcommands). Do not hand-format; the pre-commit hook enforces it.
- Version-controlled git hooks in `.githooks/`, activated via `core.hooksPath` (a
  best-effort build target sets it; fallback: `git config core.hooksPath .githooks`).
- `nuget.config`: single source (`nuget.org`) with package source mapping -- do not add a
  second feed without a matching `packageSourceMapping` entry.
- **Never build or restore with `-p:NuGetAudit=false`** to get past an `NU1902`/`NU1903`
  advisory -- that silences the check for the entire build, not just the flagged package,
  and the advisory does not self-resolve. Fix it instead: bump the vulnerable package, or,
  when it is transitive, pin the patched version directly in `Directory.Packages.props`
  plus a matching `PackageReference` in the affected project. Only when no patched version
  has shipped, suppress that one advisory with
  `<NuGetAuditSuppress Include="https://github.com/advisories/GHSA-..." />` in
  `Directory.Build.props`, with a comment saying why and what would let it be removed.
  `dotnet build` with no extra flags stays clean.
- Coverage uses the coverlet collector that ships with the test SDK, with
  `coverlet.runsettings` at the repo root (`--settings coverlet.runsettings`). It excludes
  source-generated `*.g.cs` so `[LoggerMessage]` output does not dilute the authored-code
  number; the exclusion is by file, not by attribute, because async state machines are
  `[CompilerGenerated]` and would otherwise be dropped.
- Line endings: LF everywhere except `.sln/.slnx/.ps1/.bat/.cmd` (CRLF), governed by
  `.gitattributes` and mirrored in `.editorconfig`; keep the two aligned.

## Code conventions

- One type per file -- never define multiple top-level types in one `.cs` file. File name
  matches the type name. Namespace matches folder structure (IDE0130), declared
  **file-scoped** (`namespace Foo.Bar;`), not block-scoped.
- **Default accessibility: `internal sealed`** unless a `public` surface is genuinely
  required (library entry point, or a type a framework must discover, such as xUnit test
  classes). Abstract/base classes stay `internal` (they can't be `sealed`). Interfaces are
  `public` by default. When a trusted assembly needs an `internal` type (test project,
  NSubstitute's `DynamicProxyGenAssembly2`), grant `<InternalsVisibleTo>` rather than
  widening to `public`.
- **No primary constructors** for classes and structs. Use explicit constructors with
  `_field` backing fields (`IDE0290` set to `none`; enforced by convention and review).
- **Null-guard public/internal entry points.** Every reference-type parameter of a public
  or internal constructor or method (including `this` on extension methods) is guarded
  before use: `ArgumentNullException.ThrowIfNull(x)` for objects,
  `ArgumentException.ThrowIfNullOrWhiteSpace(s)` for required strings. Guards come first,
  in parameter order. Document each with an `<exception>` tag. Exception: DI-injected
  dependencies -- trust the container, no defensive null-checks.
- Remove any DI-injected dependency that is not used in the file it is injected into.
- `CancellationToken` parameters take a default (`CancellationToken cancellationToken = default`)
  and come last, so callers may omit them. Async methods carry the `Async` suffix.
- Use **source-generated logging** (`[LoggerMessage]`) for all `ILogger` calls -- never
  `LogInformation(...)` directly (CA1873). Split large partial classes by concern:
  `Foo.cs` for logic, `Foo.Logging.cs` for `[LoggerMessage]` declarations.
- Every `catch` block that suppresses or re-routes an exception (does not re-throw) must
  emit at least one log entry at `Warning` or above -- a silent catch hides failure paths.
- Value objects: use `sealed record` (or `readonly record struct`) only for pure data
  carriers with no validation and default equality semantics. A domain value object that
  enforces invariants in its constructor is a `sealed class` implementing `IEquatable<T>`
  with matching `Equals`/`GetHashCode` and `==`/`!=` operators. Exception: a
  `readonly record struct` is fine when it wraps a single primitive, the only invariant is
  a range/null check, and structural equality is semantically correct.
- When `Equals` uses a specific `StringComparison`, `GetHashCode` must use the same
  comparer -- mismatched comparers silently break dictionary lookups.
- **Always pass an explicit `StringComparison`** to `Contains`/`IndexOf`/`StartsWith`/
  `EndsWith`/`Compare`/`Equals`/`Replace` (CA1307/CA1309). Use `Ordinal` or
  `OrdinalIgnoreCase` for identifiers, keys, and protocol values (logins, owner/repo names,
  type names); reserve the `CurrentCulture` variants for user-facing text genuinely meant
  to compare per locale. Like the comparer-matching rule above, an implicit culture-aware
  comparison is a latent bug on a non-`en-US` host, not just an analyzer nag.
- Prefer C# pattern matching over boolean chains: `x is val1 or val2`, relational patterns,
  property patterns. Always use braces for control statements. Never negate the condition
  of an `if` that has an `else` -- put the positive case first (S1940).
- When the project uses EF Core, **default reads to `AsNoTracking`**, preferably with a
  `Select` projection of only the needed columns, so nothing read-only lands in the change
  tracker. Track only when the loaded entity will be modified and saved. `FindAsync` always
  tracks and cannot be made no-tracking, so a read-only lookup is
  `Where(...).Select(...).FirstOrDefaultAsync()`, not `FindAsync`.
- Never use `!` (null-forgiving) on reflection results; use
  `?? throw new InvalidOperationException(...)`. Use `null!` (not `default!`) to suppress
  nullable warnings on uninitialized required properties.
- Keep cyclomatic complexity of any method at **15 or below**; extract helpers.
  Keep parameter counts at **7 or below** (S107); when a signature would exceed this,
  introduce a parameter object that reflects a genuine domain concept.
- No comments that restate what the code already says. Comment only the non-obvious:
  why a choice was made, a workaround, an analyzer/framework quirk, a subtle invariant.
- Naming follows the [.NET Framework Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/naming-guidelines)
  and [ASP.NET Core Engineering Guidelines](https://github.com/dotnet/aspnetcore/wiki/Engineering-guidelines#coding-guidelines)
  (ASP.NET Core wins conflicts): PascalCase types/methods/properties, `I`-prefixed
  interfaces, `_camelCase` private instance fields, camelCase parameters/locals, `var`
  wherever the compiler allows, C# type keywords (`string` not `String`), explicit member
  visibility always, `Async` suffix on async methods, no Hungarian notation or
  abbreviations in public APIs.

## Documentation

- XML docs on every `public` and `internal` type and member in non-test projects: at
  minimum a `<summary>`, plus `<param>`, `<returns>`, and `<exception>` where they add
  information beyond the signature. Complete sentences ending with a period; describe
  behavior or contract, never restate the identifier name.
- Use `<inheritdoc />` on every member that overrides or implements a documented member,
  instead of copying text; add an extra `<exception>`/`<remarks>` only when the member
  adds behavior the base does not document.
- Library projects set `<GenerateDocumentationFile>true</GenerateDocumentationFile>` so
  missing `public` docs fail the build (CS1591); `internal` members are covered by
  convention.
- Test projects are exempt -- test names document intent.

## Testing

- **TDD for production behavior:** red-green-refactor. Write a failing test first and run
  it to confirm it fails for the right reason, write the minimum code to pass, then
  refactor green. Non-behavioral edits (docs, formatting, signature tweaks) need no new
  test. For test-only changes against complete production code, satisfy failing-first with
  an `Assert.Fail("not implemented")` placeholder, confirm red, then write real assertions.
- One test class per production type, one file per test class.
- Test names use `Subject_Scenario_ExpectedResult`, all three segments PascalCase. The
  Subject segment names the observable public entry point -- not a private helper, even
  when the test reaches the helper through that entry point.
- Every test has `// Arrange`, `// Act`, `// Assert` comments. When phases collapse into a
  single expression (e.g. `Assert.Throws<>`), combine markers (`// Act / Assert`) on one
  line. Never annotate a marker with an explanation, on the same line or the next.
- No section-divider comments inside test classes -- if a divider seems necessary, the
  test names are not descriptive enough.
- Prefer `[Theory]` (`[InlineData]`/`[MemberData]`/`[ClassData]`) over near-duplicate
  `[Fact]`s. The moment a second `[Fact]` would test the same method with different
  inputs, write a `[Theory]`. `TheoryData<>` type arguments must be serializable
  primitives (xUnit1044); encode non-serializable variation as a `string` key mapped to
  setup logic in a private helper.
- **Test behavior, not implementation** -- assert observable outcomes, not internal calls.
- No trivially-true assertions: do not assert a value a query predicate already
  guarantees, and do not assert claims guaranteed by construction or type invariants
  rather than by the behavior under test.
- Coverage obligations: every `ThrowIfNull`/`ThrowIfNullOrEmpty` guard on a public or
  internal member gets at least one null/empty test; every `catch` block that suppresses
  or transforms an exception gets at least one test exercising that path; when a method
  returns a discriminated union or named result subtypes, enumerate all subtypes first
  and give each at least one dedicated test.
- Do not test `internal` helpers directly -- cover them through their public API callers.
  If a helper is unreachable through any public surface, that is a design signal.
  Internal types under test are reached via `<InternalsVisibleTo>`, not by widening
  accessibility.
- Prefer `Assert.Null`/`Assert.NotNull` over boolean-wrapped null checks (xUnit2024);
  when a test must exercise a custom `==`/`!=` null branch, use a typed null variable.
- Pass `CancellationToken.None` explicitly in tests rather than omitting the argument
  (xUnit1051). Do not add `ConfigureAwait` in test methods (xUnit1030).
- Any `IDisposable` created in a test (`HttpClient`, `HttpResponseMessage`, ...) must be
  owned and disposed (CA2000) -- test class implements `IDisposable` and tracks created
  disposables.
- `HttpMessageHandler.SendAsync` is `protected`, so NSubstitute can't intercept it -- fake
  `HttpClient` through a test handler that exposes a public abstract `MockSendAsync` and
  seals `SendAsync` to delegate to it.
- Blazor components: **bUnit + xUnit** for component tests; mock the service boundary with
  NSubstitute; simulate auth via `TestAuthorizationContext` (`bunit.web`). Do not re-prove
  database behavior through UI tests -- that is Testcontainers territory. Playwright, if
  ever, is a separate smoke-test layer in its own project.

## Configuration and secrets

- Secrets (API keys) and **connection strings** go in **user secrets**, never in
  `appsettings.json`.
- Compose passwords go in a **`.env` file**, never hardcoded in `docker-compose.yml` /
  `compose.yml`.
- Endpoint overrides go in `appsettings.json` with sensible defaults in code.
- `Properties/launchSettings.json` sets `DOTNET_ENVIRONMENT=Development`.

## Commits and pull requests

- Conventional Commits:
  `<type>(<scope>)!: <description>` where scope and `!` are optional and scope matches
  `[a-z0-9._-]+`. Types: `feat fix docs style refactor perf test build ci chore revert`.
- Body (optional): wrap at 72 characters, explain _why_ not _what_.
- AI-assisted commits carry a `Co-Authored-By: Claude <model name> <noreply@anthropic.com>`
  trailer naming the model actually in use -- never hardcode a model name in docs or
  templates.
- **After finishing a batch of work (e.g. an OpenSpec `tasks.md` batch), stop and wait for
  explicit approval before committing.** Green tests are not approval. This does not apply
  when the user's own message for that turn asks for a commit ("commit and do X") -- that
  request is itself the approval.
- PR descriptions: **under 200 words** -- what changed, why, and a short test/verification
  note. No filler phrases ("this PR...", "in this change...").
- **Always squash merge**: `gh pr merge <n> --squash`, never a merge commit or rebase, even
  when the repo allows all three. `main` stays one commit per PR, so its history reads as a
  list of shipped changes and any single entry can be reverted whole.
- Pass the squash subject explicitly (`--subject`) rather than accepting whatever GitHub
  composes from the branch's commits: the PR's conventional-commit title with the number
  appended, e.g. `feat(web): add review queue UI (#22)`. The body says what shipped and
  why, and carries the `Co-Authored-By` trailer.
- A squash-merged branch reads as **unmerged** to `git branch --merged`, because its
  commits never land on `main`. Before deleting one, confirm the content actually shipped
  with `git diff --stat main <branch>` (empty output). Do not trust `-d` to refuse: it will
  refuse a perfectly merged branch here, and `-D` alone proves nothing.
