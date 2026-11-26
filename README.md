# graphql

[![CI](https://github.com/gql-go/graphql/actions/workflows/ci.yml/badge.svg)](https://github.com/gql-go/graphql/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/gql-go/graphql.svg)](https://pkg.go.dev/github.com/gql-go/graphql)

A code-first implementation of GraphQL in Go, following the official reference
implementation [`graphql-js`](https://github.com/graphql/graphql-js).

This is a community-maintained continuation of
[`graphql-go/graphql`](https://github.com/graphql-go/graphql).

## Why this repository exists

`graphql-go/graphql` was created by [Hafiz Ismail](https://github.com/sogko) and
has been maintained for years by [Chris Ramón](https://github.com/chris-ramon).
It is a solid library that many production systems depend on, and every line of
it is their work and that of its contributors. Thank you.

The original project is in maintenance mode. Its last release, `v0.8.1`, is from
April 2023, fixes merged since then have not been released, and the GraphQL
specification has moved on by two editions. Pull requests for long-requested
features have been open for years.

This repository exists to keep the library moving:

- regular releases;
- issues and pull requests that get an answer;
- catching up with the current GraphQL specification;
- new features, added without breaking the code you already have.

It is not a rewrite and not a different library. If the original project resumes
active development, we will gladly work on bringing the two back together.

## Our commitments

**Specification compliance.** The library follows the
[GraphQL specification](https://spec.graphql.org/) and uses `graphql-js` as the
reference for behaviour. Where the library differs from the specification today,
that is a bug or a roadmap item, not a design choice.

**Backward compatibility.** Code written against `graphql-go/graphql` `v0.8.1`
must keep compiling and behaving the same way:

- changes are additive: new fields on config structs, new functions next to the
  existing ones;
- a change in behaviour is opt-in, enabled through configuration;
- the upstream test suite keeps passing unmodified, and CI rejects the removal of
  an exported symbol.

## Getting started

```bash
go get github.com/gql-go/graphql
```

The example below defines a schema with a single `hello` field and runs a query
against it.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"

	"github.com/gql-go/graphql"
)

func main() {
	fields := graphql.Fields{
		"hello": &graphql.Field{
			Type: graphql.String,
			Resolve: func(p graphql.ResolveParams) (interface{}, error) {
				return "world", nil
			},
		},
	}
	rootQuery := graphql.ObjectConfig{Name: "RootQuery", Fields: fields}
	schema, err := graphql.NewSchema(graphql.SchemaConfig{Query: graphql.NewObject(rootQuery)})
	if err != nil {
		log.Fatalf("failed to create schema: %v", err)
	}

	r := graphql.Do(graphql.Params{Schema: schema, RequestString: `{ hello }`})
	if len(r.Errors) > 0 {
		log.Fatalf("failed to execute operation: %+v", r.Errors)
	}
	rJSON, _ := json.Marshal(r)
	fmt.Printf("%s\n", rJSON) // {"data":{"hello":"world"}}
}
```

More examples are in [`examples/`](examples/).

### Migrating from `graphql-go/graphql`

The package name and the API are unchanged; only the module path is different.
Replace the import prefix across your code:

```
github.com/graphql-go/graphql  ->  github.com/gql-go/graphql
```

## What is supported today

- **Operations:** queries, mutations and subscriptions (`Do`, `Execute`,
  `Subscribe`).
- **Type system:** objects, interfaces, unions, enums, input objects, lists and
  non-null, defined in Go code.
- **Scalars:** `Int`, `Float`, `String`, `Boolean`, `ID` and `DateTime`, plus
  custom scalars.
- **Directives:** `@include`, `@skip` and `@deprecated`; custom directives can be
  declared in the schema.
- **Introspection.**
- **Validation:** the rules of the specification, plus custom rules through
  `ValidateDocument`.
- **Language tools:** lexer, parser, printer and visitor for queries and schema
  definition language (`language/`).
- **Resolvers:** `context.Context` propagation, a default resolver for structs
  and maps, lazily defined fields for circular types, and resolvers that return
  a function to run concurrently.
- **Extensions:** hooks around parsing, validation, execution and field
  resolution.
- **Query plans:** `PlanQuery`, `ExecutePlan` and `PlanCache`, to reuse the
  planning work of a repeated operation.

## Roadmap

Everything below is planned as an additive, backward-compatible change. The order
inside each group is the intended priority; it will change with feedback, so
please open an issue if something here matters to you.

### 1. Correctness

- [ ] Distinguish an explicit `null` argument from an omitted one.
- [ ] Strict serialization: an enum or scalar value that cannot be serialized
      becomes a field error instead of a silent `null`.
- [ ] Coercion of variables and default values aligned with the current
      specification.

### 2. Building blocks

- [ ] `extensions` metadata on types, fields and arguments, as in `graphql-js`.
- [ ] Field middleware: a chain around every resolver, registered on the schema.
- [ ] Executable custom directives, built on field middleware.
- [ ] Structured errors: first-class `extensions` and a hook to present or mask
      errors.
- [ ] Selection look-ahead: a simple API for a resolver to see which fields were
      requested below it.
- [ ] Access to the parent field's arguments from a child resolver.

### 3. Security

- [ ] Parser limits: maximum tokens and nesting depth.
- [ ] Validation limits: a cap on reported errors.
- [ ] Ready-made rules: maximum depth, maximum aliases, disable introspection.
- [ ] Query cost analysis, with a cost per field.

### 4. Specification catch-up

October 2021 edition:

- [ ] Interfaces implementing interfaces.
- [ ] Repeatable directives.
- [ ] `@specifiedBy` on custom scalars.
- [ ] Description on the schema definition.

September 2025 edition:

- [ ] `@oneOf` input objects.
- [ ] `@deprecated` on arguments and input fields.
- [ ] Schema coordinates.
- [ ] Descriptions on executable documents.
- [ ] Full Unicode support in the lexer.

Introspection for all of the above (`isRepeatable`, `specifiedByURL`, `isOneOf`,
`includeDeprecated` on arguments and input fields).

### 5. Performance

- [ ] Batching (DataLoader) dispatched by the executor.
- [ ] Opt-in parallel execution of sibling fields.
- [ ] Fewer allocations and less reflection in the default resolver and in list
      completion.

### 6. Go ergonomics

- [ ] Generic helpers: arguments decoded into a typed struct, typed sources,
      enums bound to a Go type.
- [ ] Built-in `JSON` and `Int64` scalars, and a companion package with common
      ones (`UUID`, `Decimal`, `Date`).

### 7. Ecosystem

- [ ] Schema utilities: print a schema as SDL, build a schema from SDL, and
      detect breaking changes between two schemas.
- [ ] HTTP transport package: GraphQL over HTTP, `graphql-transport-ws`,
      server-sent events and persisted queries.
- [ ] Apollo Federation v2 subgraph support.
- [ ] OpenTelemetry instrumentation.
- [ ] `@defer` and `@stream`, once the proposal is stable.

## Contributing

This project only works as a community effort, and help is welcome at every
level:

- **Report** a bug or a difference from the specification.
- **Tell us what you need** from the roadmap, or what is missing from it.
- **Bring over your pull request.** If you have one open on the original
  repository, open it here and it will be reviewed.
- **Review** open pull requests.
- **Become a maintainer.** Regular contributors are invited to join the
  organization; the goal is for this project never to depend on a single person.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the guidelines.

## License

[MIT](LICENSE), the same as the original project.
