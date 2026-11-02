---
name: rules-cascade-architecture
description: Explain how Rule Cascade is designed and how a project uses it (rulesets, bundles, engines, enforcement, inheritance, rcas). Use when the user asks how Rule Cascade works, why it is built a certain way, which engine form to use (library, command, WebAssembly, rule server), how rules reach a browser or a service, or how to set it up in a project.
metadata:
  internal: true
---

# How Rule Cascade works

Business rules are written once, as data, and evaluated identically by Python, TypeScript, Java,
Go, the `rcas` command and WebAssembly. Enforcing a rule never depends on a shared service being up.
Answer design questions from this page, and read the linked pages for detail before you advise on
a production setup.

## The pipeline

| Stage | Input | Output |
|---|---|---|
| Author | A change request | `rules/*.ruleset.yaml` with golden tests, in a pull request |
| Check | The ruleset, its parent, the entity schemas | `rcas check`: schema, load checks, golden tests. CI runs it |
| Compile | The same | One JSON bundle per ruleset in `build/rules`, identified by a SHA-256 checksum |
| Evaluate | A manifest and a request | Findings, field states, computed values, a decision, commands |

A bundle holds two manifests. The **server manifest** has every rule. The **client manifest** has
only the rules with `enforcement: client` or `both`, and is public by design. Evaluation is a pure
function: no clock (the host passes `ctx.now`), no network, no files. Two engines given the same
bundle and request return the same result; a conformance suite checks it on every build.

## Which engine form

| Form | Use it when |
|---|---|
| Native library (`@rules-cascade/core`, `rule-cascade` on PyPI, `com.rulescascade:rules-cascade-core`, `rulescascade.com/go`) | The language has one: evaluation is a function call, no hop |
| `rcas engine` (JSON Lines on stdin/stdout) | Any other language that may start a child process |
| `rcas.wasm` (WASI preview 1) | No native binary can be shipped or started |
| Rule server (`@rules-cascade/server`, HTTP) | The caller cannot run an engine; it never replaces the check in the service that owns the data |
| Browser (TypeScript runtime, client manifest) | Instant feedback while the user types; advice only |

Prefer the form closest to the caller. Custom operators exist only in native libraries, registered
by the host; the command, the module and the stock server have none.

## The rules that follow from the design

- **The server decides.** Every state-changing operation is evaluated on the server, whatever the
  browser said. The actor comes from the host's authentication, never from the request.
- **What must stay secret is `enforcement: server`.** Anything a client rule uses reaches every user.
- **Everything fails closed.** A ruleset that fails a check produces no bundle; a rule that cannot be
  evaluated is a blocking `RULE-EVALUATION-ERROR` and the decision is `deny`.
- **One parent per ruleset.** Levels form a chain (organization, module, feature ...). The parent's
  `overridePolicy` (`locked`, `tighten-only`, `open`) says what a child may change; the compiler
  flattens the chain, so depth costs nothing at run time.
- **Versions are immutable.** Log the checksum with every decision; cache bundles without expiry.
- **Rules change like code.** Pull request, `rcas check` in CI, compile, roll out or reload. Agents
  propose (`propose_ruleset`, `rcas proposals accept <id>`); they never write rulesets directly.

## Design decisions

| Decision | In one line |
|---|---|
| Expressions are a JSON tree of `{op, args}` | Portable, analysable, no eval of code |
| Embedded runtimes first | No shared service on the request path |
| Decimal arithmetic inside | Doubles in, decimals inside, 15 significant digits out, same everywhere |
| Single inheritance with override policy | Parents decide what children may change |
| Compile once into bundles | Evaluators stay small; compilers run in CI |
| Portable patterns | `matches` accepts one regex subset with one meaning in every language |
| Engine as a program | JSON Lines protocol, one static command, one WASI module |
| Functions and custom operators | Reuse inside the contract; host operators as the escape hatch |
| Logical places and data types | Rules target fields and semantic types, not screens or columns |
| Version checks at load | A ruleset needing a newer runtime fails at load, not per request |

The records: https://rulescascade.com/reference/architecture/ and the ADRs linked from it.

## Using it in a project

1. Install `rcas`: `npm i -g @rules-cascade/cli`, or https://rulescascade.com/get-started/install/.
2. `rcas init` writes `rcas.yaml` (rules dir, output dir, languages, sources), a first ruleset with
   golden tests, and optionally the agent files and the MCP configuration.
3. Find what to move: `rcas analyze` (code), `rcas derive` (OpenAPI or JSON Schema baselines).
4. Write and check: the `rules-cascade` skill. Review: `rules-cascade-review`.
5. `rcas compile --all` in CI; ship the bundles with the application.
6. Enforce: the skill for the stack (`rules-cascade-typescript`, `-python`, `-java`, `-go`,
   `-engine`). Replace old checks step by step with `rules-cascade-migrate`.
7. Agents: `rcas mcp install` (tools) and `rcas agent install` (instructions, skills, sub-agents).

More: https://rulescascade.com/reference/specification/, https://rulescascade.com/reference/enforcement-guide/,
https://rulescascade.com/reference/caching/, https://rulescascade.com/agents/.
