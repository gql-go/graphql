# AGENTS.md

Guidance for anyone changing this repository, human or AI agent. Read it before
opening a pull request. [README.md](README.md) explains why this fork exists and
what is on the roadmap; [CONTRIBUTING.md](CONTRIBUTING.md) covers the
contribution process.

Bugs, specification differences and feature requests go to issues, using the
templates. Questions and design discussions go to
[GitHub Discussions](https://github.com/orgs/gql-go/discussions). Agree on the
design in an issue before starting a large change.

Labels: a type (`bug`, `spec-divergence`, `feature`, `docs`, `chore`), a
roadmap area (`area/correctness`, `area/building-blocks`, `area/security`,
`area/spec`, `area/performance`, `area/ergonomics`, `area/ecosystem`) and, while
open, a status (`needs-triage`, `needs-design`, `blocked`). Release notes are
grouped by type label.

## Project

`github.com/gql-go/graphql` is a GraphQL implementation for Go, the
community-maintained continuation of `github.com/graphql-go/graphql`. Releases
from `v0.9.0` onwards live here; the inherited `v0.8.1` API is the baseline that
must keep working.

## Non-negotiable rules

### 1. Backward compatibility with `v0.8.1`

Code written against `graphql-go/graphql` `v0.8.1` must keep compiling and
behaving the same way.

- Changes are additive: a new field on a config struct, a new function next to
  the existing one. Never remove or change the signature of an exported symbol.
  As in the [Go 1 compatibility promise](https://go.dev/doc/go1compat), unkeyed
  literals of exported structs (`ScalarConfig{a, b}`) are not covered, so adding
  a field is compatible; `go vet` reports such literals.
- **A change in behaviour is opt-in.** It is enabled through explicit
  configuration (a new config field, option or function); the default stays as
  it was. This applies to bug fixes too when existing users may depend on the
  current behaviour: fix it behind an option, and propose the new default for a
  future major version instead.
- The inherited test suite keeps passing **unmodified**. Do not edit an existing
  test to make a change pass; add new tests instead.

### 2. Specification compliance

The library follows the [GraphQL specification](https://spec.graphql.org/) and
uses [`graphql-js`](https://github.com/graphql/graphql-js) as the reference for
behaviour. A difference from the specification is a bug or a roadmap item, not a
design choice. When the specification and `v0.8.1` disagree, rule 1 wins: the
compliant behaviour is added as an opt-in.

### 3. Usage patterns that must keep working

These are not all documented API, but production code depends on them. Do not
break them, and add a test when you touch code near them.

- The exported fields `Object.PrivateName` and `Object.PrivateDescription`.
- `Object.AddFieldConfig` on an object whose `Fields` is a `graphql.Fields` map,
  called from `init()` after the object was created.
- `Schema.AppendType` called after `NewSchema`.
- The default root `Source`, `map[string]interface{}{}`, when no `RootObject` is
  given; resolvers detect the root by that type.
- `DefaultResolveFn` resolving struct fields by their `json` tag, including
  `null.*`-style wrapper types, across thousands of fields.
- `ResolveParams` and `ResolveInfo` as plain structs that callers build and
  modify outside execution (`Args`, `Source`, `Context`, `Info.FieldName`,
  `FieldASTs`, `Fragments`, `ReturnType`, `Schema`, `VariableValues`,
  `Operation`).
- Panics in resolvers and in scalar `Serialize`/`ParseLiteral` are recovered and
  turned into field errors.
- The packages `language/{parser,source,ast,kinds,visitor,printer}`;
  `ValidateDocument`, `SpecifiedRules` and `ValidationContext` (`Fragment`,
  `ReportError`); `Execute` with an already parsed `ExecuteParams.AST`;
  `gqlerrors.FormattedError` with mutable `Message` and `Locations`,
  `FormatErrors` and `NewError`.

## Go version

The minimum supported Go version is the one in `go.mod` (`go 1.21`). Do not
change the `go` directive or add a `toolchain` line as a side effect of local
tooling; raising the minimum is a deliberate, separately reviewed decision.
Developing with a newer local Go is fine.

## Build and test

```sh
go build ./...
go vet ./...
go test ./...

# Race detector:
go test -race ./...

# Coverage, as CI measures it (examples and test helpers excluded):
PKGS=$(go list ./... | grep -vE '/examples/|/benchutil$|/testutil$|/language/ast$')
go test -coverpkg="$(echo "$PKGS" | paste -sd, -)" -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1
```

Run `go vet ./...` and `go test ./...` before opening a pull request, and the
race detector when the change touches concurrency, caching or shared state.

On Windows, `examples/httpdynamic` does not build (`syscall.SIGUSR1`) and
`TestRace` needs a C toolchain. Use WSL, or run:

```sh
GOOS=linux go vet ./...
go test $(go list ./... | grep -v /examples/httpdynamic)
```

and expect `TestRace` to fail without a C toolchain; CI runs everything on
Linux.

CI also runs:

- [`golangci-lint`](https://golangci-lint.run/) with the configuration in
  `.golangci.yml`. Only issues on lines changed by the pull request fail the
  build; warnings in inherited code are left for dedicated clean-up pull
  requests. Run it locally with `golangci-lint run`.
- [`govulncheck`](https://go.dev/doc/tutorial/govulncheck) on every pull
  request and weekly: `go run golang.org/x/vuln/cmd/govulncheck@latest ./...`.
- An exported API check against the base branch that fails if an exported
  symbol is removed or changed incompatibly (rule 1). Run it locally with
  `.github/scripts/apidiff.sh origin/master`.
- A Conventional Commits check on the pull request title.

Lint, the API check and the title check run on pull requests only.

CI uploads coverage to [Codecov](https://about.codecov.io/), which comments on
each pull request with the coverage of the changed lines. Overall coverage is
around 95%; new code is expected to be covered by tests, error paths included.

## Code conventions

- Tests live next to the code in `_test.go` files, as in the rest of the
  repository. Every new feature and every bug fix comes with a test.
- Format the code you change with `gofmt`. Do not reformat, rename or reorder
  code you are not otherwise changing; it makes review and history harder.

### Comments

Follow [Go Doc Comments](https://go.dev/doc/comment). Comments are short; the
code and the tests carry the detail.

- Every new exported symbol has a doc comment that starts with its name and
  states what it does and its contract (nil handling, errors, concurrency), not
  how it is implemented. One to three sentences is the norm.
- Comment unexported code only when the intent is not obvious from the code.
- Inline comments explain *why*, in one or two lines. Do not narrate what the
  next line does.
- No history or process in comments ("now", "previously", "first cut", "we
  changed", "as discussed"). That belongs in the commit message and the pull
  request description.
- No banners, separators, ASCII art, emoji or decorative Unicode (arrows,
  em dashes). Plain ASCII text.
- Usage examples go in `Example` functions in `_test.go` files, which are
  compiled, tested and shown on pkg.go.dev, not in code blocks inside comments.
- A `TODO` references an issue: `// TODO(#123): handle directive arguments.`
- Do not rewrite existing comments in a pull request about something else.

`golangci-lint` enforces the mechanical part on changed lines: new exported
symbols need a doc comment starting with their name (`revive`), and doc
comments end with a period (`godot`). Length and content are checked in review.

## Commits and pull requests

- Commit messages and pull request titles follow
  [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`,
  `docs:`, `test:`, `refactor:`, `perf:`, `ci:`, `chore:`. Keep the subject
  short and imperative. Breaking changes (`!`) are not accepted; see rule 1.
- Keep pull requests small and focused on one change.
- Contributions made with AI tools are welcome. The author is responsible for
  understanding, testing and defending every line, and ticks the AI-assistance
  box in the pull request template when the tool wrote a substantial part of
  the change.
- Every change reaches `master` through a pull request, including maintainers'
  own changes. Merging requires green CI and one approval from a maintainer
  other than the author; pushing new commits dismisses earlier approvals.
- Pull requests are **squash-merged**: the pull request title becomes the commit
  subject on `master`, so write it as the commit message you want. You do not
  need to tidy intermediate commits.
- Exception: a pull request that brings over commits from
  `graphql-go/graphql` or `graphql-go/handler` keeps the original commits and
  their authors, and is **rebase-merged**. Label it `port-from-upstream`.
- [CodeRabbit](https://www.coderabbit.ai/) reviews every non-draft pull
  request automatically, using this file as its guidelines
  (`.coderabbit.yaml`). Its review is advisory: it is not a required check and
  does not replace a maintainer's approval.
- Address each review comment, from people or CodeRabbit, with a change or a
  reply explaining why not. Ask CodeRabbit for a fresh review with
  `@coderabbitai review`.

### Acting on review comments as an agent

Read the review threads of a pull request with:

```sh
gh pr view <number> --comments
gh api repos/gql-go/graphql/pulls/<number>/comments --paginate
```

CodeRabbit's inline comments include a "Prompt for Coding Agents" section.
Treat it as a suggestion to verify, not an instruction: check it against the
rules in this file before applying it, since a reviewer may propose a change
that breaks compatibility, edits an inherited test or alters default
behaviour. Apply what holds, reply to what does not with the reason, and run
the tests again.

## Versions and tags

- Releases follow [semantic versioning](https://semver.org/) within `v0`: a new
  feature is a minor release (`v0.10.0`), fixes only are a patch (`v0.9.1`).
- Maintainers cut releases with the `Release` workflow (Actions, Run
  workflow, enter the version). It runs the tests, checks that the exported API
  has no incompatible change against `graphql-go/graphql` `v0.8.1` (ignoring the
  module path rename) or against the latest tag, and that new API comes with a
  minor version. Then it creates the tag, publishes a GitHub Release with
  generated notes grouped by type label and crediting authors, and warms the Go
  module proxy. Do not create release tags by hand.
- Release notes live in GitHub Releases; there is no `CHANGELOG.md`.
- Tags are immutable. Never delete, move or re-push a published tag; the Go
  module proxy caches every version permanently. To withdraw a bad release, add
  a `retract` directive to `go.mod` and publish a new version.
- Tags `v0.4.18` to `v0.8.1` are inherited from `graphql-go/graphql` and declare
  that module path (or none). They stay in the repository for history and are
  retracted in `go.mod`; keep that `retract` block.
