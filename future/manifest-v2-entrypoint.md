# Python modules as manifest v2 entrypoints

author: yves
created: 2026-09-03
status: draft, probed on the merged engine, reviewed three times, waiting for design approval before implementation
related: `dagger/dagger#14038` (manifest v2, branch `manifest-v2`, head
`75c777223ccc4baaf5819a04d70d060034a94dbb`); `dagger/dagger#13992` (SDK UX
"module-max", branch `sdk-ux-module-max`, head
`2dfc08f72c87ba5b47f1aa47752fea605fdace01`); `dagger/java-sdk#19` and
`dagger/go-sdk#36` (prototypes of the same adaptation for other SDKs);
`dagger/python-sdk#22` (clients) and `dagger/python-sdk#24` (embedded Dang
runtime), both open; `future/done/self-contained-python-sdk.md`;
`hack/designs/2026-08-25-python-module-performance-ideas.md`.

## Summary

Two draft engine pull requests change how a module is generated and how it is
loaded. Neither has been run against a real SDK. This document adapts
`dagger/python-sdk` to both, so that a Python module can be created with
`dagger module init python`, generated with `dagger generate`, and called with
`dagger call`, on an engine that carries both changes.

The design keeps everything that works on the released engine
(`v1.0.0-beta.11`) working, with one narrowed behaviour stated in section 0,
and makes the authoring module load on both engines. It adds a second,
parallel path for the new engine. The two paths share the code that builds a
Python module's container.

## Problem

### What the engine changes

**Manifest v2 (`dagger/dagger#14038`).** Today the engine loads a Python module
by calling this repository's runtime module, which returns a `Container`. The
engine then execs that container twice per call: once with an empty function
call to discover the module's types, once with the real call. The container's
Python process reads the call from `Query.currentFunctionCall` and writes the
result with `FunctionCall.returnValue`. This is an implicit protocol.

Manifest v2 replaces it. The module's `dagger-module.toml` becomes:

```toml
manifestVersion = 2
name = "hello"

[entrypoint]
kind = "dang"
source = "./sdk/entrypoint"
```

The engine loads the Dang program in `entrypoint.source` directly. That program
must define exactly one type that implements this interface:

```graphql
interface ModuleEntrypoint {
  types(workspace: Workspace!): [TypeDef!]!
  call(
    workspace: Workspace!
    receiverType: String!
    receiverValue: JSON
    fnName: String!
    fnArgs: JSON!
  ): JSON!
}
```

The engine calls `types` to install the module and `call` for every
constructor or function invocation. No SDK module is called at load or call
time. The SDK's only job is to generate the entrypoint. The interface has no
notion of a main type: `types` returns every type, and the engine exposes the
one type that has a constructor. The engine at head `75c77722` errors when more
than one returned type has a constructor.

**SDK UX (`dagger/dagger#13992`).** The engine stops calling the beta
SDK-module interface this repository implements today. It deletes
`Query.currentModule.asSDK` (`core/current_module_as_sdk.go` and
`core/schema/module_as_sdk.go` are removed) and
`ModuleSource.generateLocalDependencies`, and it replaces the `dagger module
init` flow that called `initModule` and `targetRuntime`. An SDK module is
instead an installed module that implements:

```graphql
interface Sdk {
  detectScope(ws: Workspace!): String!
  generateScope(ws: Workspace!, isModule: Boolean!, name: String!, clients: [ModuleSource!]!): Workspace!
}
```

`dagger module init python --name=hello` records a scope in `dagger.toml` and
calls `generateScope` with `Workspace.cwd` set to the module directory.
`dagger generate` calls it again for every recorded scope. The SDK writes
`dagger-module.toml` itself. The workspace records SDKs as `[sdks.python]
module = "python-sdk"` and scopes as `[sdks.python.scopes."<path>"]`, not as
`[modules.python-sdk.as-sdk]`.

### What this means for this repository

Dang type-checks a whole module against the engine's schema when the module
loads. A module that selects a field the engine does not have fails to load;
no function of it can be called. Every public function of the authoring module
selects `currentModule.asSDK` (`python-sdk.dang`: `modules`, `mod`,
`managedModuleAtOrAbove`, `generateAll`), and `Mod.generate` selects
`generateLocalDependencies` (`mod.dang`). On an engine with `#13992` the
authoring module therefore does not load at all, so `detectScope` and
`generateScope` could not be reached even if they were added. The runtime
module in `runtime/` implements a contract (`moduleRuntime(modSource):
Container`) that `#14038` retires for new modules.

This is also the first time either engine change meets a real SDK. Its author
says so in the pull request descriptions. Part of the deliverable is a list of
what does not work, with evidence.

## Goals

1. On an engine with both `#14038` and `#13992`: `dagger module install
   <python-sdk>`, `dagger module init python --name=hello`, `dagger generate`
   and `dagger call hello <function>` work for a Python module in the caller's
   workspace, through a generated Dang entrypoint. Constructor arguments,
   function arguments, objects returned by functions, enums and raised
   exceptions behave as they do today, within the limits listed under
   *Accepted differences*.
2. The generated `types()` derives the type definitions from the module's real
   Python classes, with the same rules the runtime applies today. A change to
   a Python function signature is visible on the next `dagger call` without
   re-running `dagger generate`.
3. Everything that works on `v1.0.0-beta.11` keeps working: `initModule`,
   `targetRuntime`, `mod`, `modules`, `generateAll`, the runtime module, and
   the e2e checks that this repository's CI runs on the released engine. The
   authoring module loads on both engines. One behaviour is narrowed, stated
   and covered in section 0: `generate` no longer generates a module's
   ungenerated local dependencies on its behalf. This narrowing is the one
   open decision in this document (see section 0).
4. Report every gap found in the two engine changes to their author, with a
   reproduction, and report nothing that has not been reproduced.

### Accepted differences

These are consequences of the `ModuleEntrypoint` interface as specified. They
are acceptance limits of this change, not defects in it:

- A function whose cache policy is `Never` or `PerSession` is still cached
  by the content of its inputs, because the entrypoint runs a plain container
  exec and receives no per-call nonce or policy.
- A function error reaches the user as an exec failure with the process's
  stderr. The structured values today's runtime attaches to a `dagger.Error`
  (exception type, stacktrace, exec details) are lost: probe 1 showed that
  `Query.currentFunctionCall` is reachable inside the Dang program but not
  inside its nested exec, so the Python process has no `returnError` to
  call.
- The module docstring (`Module.withDescription` today) is not exposed.

## Non-goals (YAGNI)

- Loading a v2 Python module from a git ref, or from any workspace other than
  the one that contains its source. The entrypoint receives the caller's
  workspace (see *Verified constraints*), and a module that is not in that
  workspace cannot find its own files. This is an engine gap, reported below.
- An engine that carries `#13992` but not `#14038`. `generateScope` writes a
  v2 manifest that such an engine cannot load, and that engine has no CLI
  flow that reaches `initModule`. That combination is transient and
  unsupported.
- Generating clients for other modules inside `generateScope`. That is
  `dagger/python-sdk#22`, an open series with its own design. `generateScope`
  rejects a non-empty `clients` list with an error that names that work.
- Generating a module's local dependencies as part of `generate`, on any
  engine. See section 0 and `dagger/python-sdk#22`, which designs recursive
  generation by ownership.
- Removing the v1 runtime module or the beta SDK-module functions. They are
  what the released engine runs. Removal is a follow-up for after an engine
  release ships both changes.
- The `module` entrypoint kind (an entrypoint that is itself a module). The
  engine at head `75c77722` returns "not implemented" for it.
- Implementing `defaultModulePath`. The engine default
  (`.dagger/modules/<name>`) is what `dagger module init` produces today.
- Static type definitions written into the entrypoint at generate time. See
  *Alternatives*.
- Migrating an existing v1 (`[runtime]`) module to v2 automatically.
  `generateScope` keeps a v1 manifest it finds and regenerates only `sdk/`.
- The `legacy` template on v2. It scaffolds a `dagger.json`-era module whose
  `.gitignore` ignores `/sdk`, which a v2 module must commit.
  `generateScope` rejects `template = "legacy"`; `initModule` keeps it.
- Supporting a manifest v2 module whose `pyproject.toml` references paths
  above the module directory. The v2 manifest has no `include` list, so the
  entrypoint mounts only the module directory. Reported below.
- Keeping this repository's own `dagger.toml` valid for both engines. It stays
  in the beta form so that CI keeps running; the merged-engine tests use a
  scratch workspace in the `[sdks.python]` form.

## Verified constraints

All engine references are to `dagger/dagger` at
`75c777223ccc4baaf5819a04d70d060034a94dbb` (manifest v2) or
`2dfc08f72c87ba5b47f1aa47752fea605fdace01` (SDK UX). The two branches share
merge base `e3a8e7eb858fc34186c5521139aa7524cba9b41d` on `main`.

**The merged engine.** The probes in *Testing* run against an engine built
from a merge of the two heads. The recipe, reproducible from the two SHAs:
check out `75c77722`, `git merge 2dfc08f7`; the only conflict is in
`dagger.toml`, where each side adds one `[modules.<name>]` table (`tiny` and
`tla-check`); keep both. The resulting tree is
`d610a2004e4e146795ee356e602e79e2f341c276` (a tree hash, which any run of the
recipe reproduces; the merge commit itself is local). The engine image is
built with the repository's own `dev` module (`.dagger/modules/dev`,
`deploy`, which is what `hack/build` runs) and the CLI with
`.dagger/modules/cli-dev` (`binary`); `hack/build` itself stalls on this
merge because the pinned CLI it drives loads every workspace module of the
repository first.

**What the SDK UX branch removes and keeps** (checked by reading the schema
registrations in `core/schema/*.go` at `2dfc08f7`):

- Removed: `CurrentModule.asSDK` (the `dagql.Fields[*core.CurrentModule]` list
  in `core/schema/module.go` has `dependencies`, `generatedContextDirectory`,
  `name`, `source`, `workdir`, `workdirFile`, `generators`);
  `ModuleSource.generateLocalDependencies` (the `NodeFunc` registration is
  deleted from `core/schema/modulesource.go`). `Query.moduleManifest` is
  added, not removed.
- Kept, with the same names on `v1.0.0-beta.11`: `Workspace.sdk(name)`,
  `Workspace.sdks`, `WorkspaceSDK.modules { name source }`,
  `Workspace.findUp(name, from)`, `Workspace.withWorkdir`,
  `Workspace.withNewFile`, `Workspace.withDirectory`, `Workspace.withChanges`,
  `Workspace.changes`, `Workspace.cwd`, `Workspace.moduleSource`,
  `ModuleSource.introspectionSchemaJSON`,
  `ModuleSource.generatedContextDirectory`, `ModuleSource.sourceSubpath`,
  `ModuleSource.digest`, `ModuleSource.configExists`,
  `ModuleSource.moduleOriginalName`, `ModuleSource.contextDirectory`,
  `ModuleSource.sdk`, `CurrentModule.name`, `CurrentModule.source`,
  `Query.changeset`, `Changeset.withChangesets`. Every selection in
  `python-sdk.dang`, `mod.dang`, `mod-config.dang`, `template.dang` and
  `runtime/main.dang` other than the two removed ones is in this list.
- `Workspace.sdk(name)` accepts the SDK's installed module name as well as
  its SDK name on both engines. On `v1.0.0-beta.11` (checked on 2026-09-03 in
  this repository's workspace) `sdk(name: "python-sdk")` and `sdk(name:
  "python")` both return the nine registered modules, each with `source` as a
  workspace-root-relative path. `name` differs between the engines: the
  released engine returns the directory's base name
  (`core/schema/workspace_sdk.go`, `filepath.Base(source)`), the SDK UX
  branch returns the scope's recorded name. The design reads only `source`.
  On `2dfc08f7`, `workspaceSDKFromEntry` builds the list from
  `[sdks.<name>.scopes]` entries with `is-module = true`, and
  `installedSDKSource` resolves a module name through `SDKNameForModule`.
- Neither engine narrows the list to the caller's `cwd`. The narrowing that
  `currentModule.asSDK` did on the released engine
  (`core/schema/module_as_sdk.go`, `currentModuleAsSDKModulesForCwd`) is:
  deduplicate by cleaned path; with `cwd` at the root, return every module;
  otherwise return every module at or below `cwd` in configuration order,
  preceded by the nearest module that strictly contains `cwd` unless `cwd`
  is itself a module. `docs/cwd-aware-discovery.md` describes the same
  policy.
- This repository's `dagger.toml` still parses on the SDK UX branch:
  `workspace.ParseConfig` ignores unknown keys, so `[modules.python-sdk.as-sdk]`
  is dropped and no SDK is registered. That is enough for the authoring
  module to load; it is not enough for `dagger module init python`, which
  the merged-engine tests run from a scratch workspace.
- Probe 0 (merged engine, 2026-09-03): the authoring module at
  `2482284f` *loads* (`dagger functions` lists its functions), because the
  engine's type-definition phase declares Dang signatures without inferring
  bodies; every *call* then fails with "field `generateLocalDependencies`
  not found in Dagger.ModuleSource" (and the `asSDK` selection behind it),
  because a call infers the whole program. So section 0 is what makes the
  module callable, not loadable.

**How the engine drives an entrypoint** (`core/sdk/dang/v2/entrypoint.go`):

- `resolveEntrypointSource` resolves `entrypoint.source` relative to the
  directory holding `dagger-module.toml`, from the module's own context
  directory. A value starting with `.` is a local path; a value starting with
  `/` is classified local and then rejected as absolute by
  `entrypointSourceSubpath`.
- The `workspace` argument passed to both `types` and `call` is
  `Query.currentWorkspace`, selected on the engine's server and passed by ID
  to the entrypoint's nested client. Probe 1 (merged engine, 2026-09-03):
  `workspace.cwd` is `/` whether `dagger call` runs from the workspace root
  or from inside the module directory, and `workspace.directory("/")` is
  readable from the nested client (its entries are the workspace root's).
  Nothing sets `cwd` to the module directory, so the entrypoint has no
  runtime way to find its module other than a path baked at generate time.
- `runEntrypointDir` copies every `.dang` file in the source directory into one
  temporary program, appends the `ModuleEntrypoint` interface as
  `__module_entrypoint.dang` (that file name is therefore reserved), and runs
  the whole program with `dang.RunDir` on every `types` and every `call`. A
  generated entrypoint can be split across files; everything in them is
  re-evaluated per call.
- `findModuleEntrypoint` requires exactly one public type implementing
  `ModuleEntrypoint`, constructible with zero arguments. Private `let` fields
  with defaults are not constructor arguments (dang v2.1.3
  `pkg/dang/fields.go`). Other types in the program are allowed.
- `types` results are read as a Dang list of `TypeDef` values; each element's
  ID is loaded on the engine's server with `dagql.ID.Load`. A `TypeDef` an
  entrypoint returns must therefore be a value the Dang program built
  itself: the schema the engine serves to modules exposes no `load*FromID`
  field (its `Query` type has thirty fields; `node` is the only ID loader),
  Dang exposes no way to call `node` with a type, and `JSON.decode` into a
  `TypeDef` fails with "expected object for TypeDef, got string" (probe 1).
  A nested process cannot hand type definitions to the entrypoint by ID.
- `call` results must be a Dang `JSON` scalar (or null). The engine passes
  `receiverValue` and `fnArgs` to the entrypoint as `JSON` scalars
  (`jsonScalarCallArg`). `fnArgs` is one JSON object, argument name to
  embedded JSON value; `receiverValue` is the receiver's JSON state, `{}`
  for a function on a freshly constructed object, null for a constructor. A
  raised Dang error is the function error. Probe 1: `JSON.encode` of the
  four-member request record produces, with keys sorted,
  `{"fnArgs":"{\"who\":\"world\"}","fnName":"hello","receiverType":"Probe","receiverValue":"{}"}`.
- The entrypoint program runs as a nested client
  (`dangshared.WithNestedClientServer`) with the module as context and, for
  `call`, with the engine's real `FunctionCall` bound (`shared.go`:
  `ServeHTTPToNestedClient(..., moduleContext, fnCall)`). So
  `Query.currentFunctionCall` resolves inside the Dang program during `call`
  (probe 1: `currentFunctionCall.name` is the function name). Inside a
  container exec started from the program with `experimentalPrivilegedNesting:
  true` it does not: the query fails with `get field "name": reflect: call of
  reflect.Value.Field on zero Value`. The exec itself can build `TypeDef`s
  through the API during `types` and `call`, and an exec whose inputs are
  unchanged is reported `CACHED` by a later CLI invocation.

**How the engine drives an SDK module** (`core/sdkmodule/provider.go`,
`core/schema/workspace_sdk_module.go`):

- `sdkmodule.Load` loads the module with `asModule` first, then validates the
  exact shapes and argument order of `detectScope(ws: Workspace!): String!`
  and `generateScope(ws: Workspace!, isModule: Boolean!, name: String!,
  clients: [ModuleSource!]!): Workspace!` on the main object.
  `defaultModulePath` is optional. Extra functions are ignored, but only once
  the module has loaded.
- `resolveSDKModuleInit` rejects only an empty module name. Quotes,
  backslashes and control characters in a name or path reach the SDK
  unchanged.
- `workspaceAtSDKModuleScope` calls `generateScope` with `ws.cwd` set to the
  scope path, after creating the directory if it does not exist. The
  `Workspace.cwd` field returns that path in API form
  (`core/schema/workspace.go`, `workspaceAPIPath`): `/` for the root and a
  leading `/` otherwise, so a scope at `.dagger/modules/hello` reads as
  `/.dagger/modules/hello`.
- `validateSDKModuleWorkspace` rejects a result whose `cwd` changed or that
  touched `dagger.toml`. `validateGeneratedModuleConfig` requires a parseable
  `dagger-module.toml` at the scope when `isModule` is true.
- Provider constructor arguments are the SDK's settings. The CLI exposes them
  as flags of `dagger module init python`. Explicit settings are persisted on
  the scope and the provider is constructed with them on every later
  generation.
- `Query.moduleManifest.v2(name).withDangEntrypoint(source).asFile` writes a v2
  manifest. It does not exist on `v1.0.0-beta.11`.

**Released engine behaviour that the design relies on** (checked against
`v1.0.0-beta.11` on 2026-09-03):

- `Workspace.moduleSource(path).introspectionSchemaJSON` on a directory that
  holds only a `manifestVersion = 2` manifest succeeds (`configExists: true`,
  350,499 bytes). The same call on a directory with no manifest fails with
  "dir module source does not contain a dagger config file". So the manifest
  must be written before bindings are generated, and it can be the v2 one.
  The released engine reads no `engineVersion` from a v2 manifest and serves
  its oldest schema view (`engine.MinimumModuleVersion`, `v0.9.9`), in which
  `Workspace.cwd` and the other `v1.0.0-0`-gated fields are absent; the
  merged engine sets the version to its own (`core/schema/modulesource.go`,
  `initFromModConfig`), which is the view the module will run against. A
  module source whose manifest declares `engineVersion = "v1.0.0-0"` (the
  runtime module's does) yields the current view on the released engine.
- `Container.withExec(stdin:, experimentalPrivilegedNesting:)`,
  `Directory.exists` and `Directory.digest` exist on both engines.
  `Query.loadTypeDefFromID` exists in the core schema of both but not in the
  schema the engine serves to modules (probe 1).

**Dang** (`github.com/vito/dang/v2` v2.1.3, the version the engine embeds):
`JSON.encode`, `JSON.decode`, `(x :: JSON!)`, string `split`/`trimSpace`/
`replace`/`contains`/`containsMatch`, list `map`/`filter`/`reduce`/`+`,
`raise` and `rescue` are available. `JSON.encode` of a record whose member is
a `JSON` scalar writes that member as a JSON text string, not as embedded JSON
(`tests/test_codec.dang`). `JSON.decode` materialises records, lists,
enums and scalars against a `:: T` hint, and a record type may contain lists
of other record types (`tests/test_from_json.dang`); it cannot materialise a
GraphQL object. Probe 2 (merged engine, 2026-09-03, in an entrypoint program)
also decoded a top-level list of records (`:: [Rec!]!`), a record type that
refers to itself (`type Node { children: [Node!]! }`), and a nullable record
member absent from the input (read back as `null`). Methods may recurse
(`tests/test_cow_recursive.dang`). `loadTypeDefFromID` is "not found" in an
entrypoint program (probe 1).

## Proposed approach

### 0. One authoring module that loads on both engines

The beta functions stay, but they stop selecting the two fields the SDK UX
branch removes.

**Module discovery.** `modules(ws)` and `mod(ws, path)` read `ws.sdk(name:
currentModule.name).modules`, which exists on both engines, instead of
`currentModule.asSDK(workspace: ws).modules`. The cwd narrowing the engine did
for `asSDK` moves into Dang, rule for rule as quoted in *Verified
constraints*, with the helpers `python-sdk.dang` already has (`normalizePath`,
`pathDepth`, `pathContains`). `mod` returns the nearest module containing the
requested path or raises. The existing discovery checks plus a new fixture
with a registered module nested inside another registered module (see
*Testing*) are the regression net. `docs/cwd-aware-discovery.md` and the
docstrings of `modules` and `managedModuleAtOrAbove` change to say that the
engine owns membership and the SDK applies the cwd policy.

**Local dependencies.** `Mod.generate` stops calling
`generateLocalDependencies`. On the released engine that call generated a
module's local dependencies into a staging workspace before reading the
module's schema, so that a dependency with no committed bindings did not make
the module's generate fail. Nothing in `generateAll` replaces it: each module
generates from the same base workspace and the changesets are merged, so one
module never sees another's fresh output. After this change, a module whose
local `dagger-module.toml` dependency has no committed bindings fails to
generate until the dependency is generated and committed first; a module
whose dependencies are generated is unaffected. This is a narrowing of goal
3, chosen because the only alternative is to re-implement recursive
generation in Dang, which `dagger/python-sdk#22` already designs
("dependencies registered to this SDK are generated recursively and
overlaid") as part of turning dependencies into clients. Doing it twice is
the coordination cost this document avoids. The narrowed behaviour is
covered by `generateDependencyOrderCheck` (*Testing*), which also shows the
two-step path a user takes.

**Decision requested.** The plan reviewers split on this point: one accepts
the narrowing as stated, the other holds that goal 3 as originally settled
("everything on the released engine keeps working") forbids it and that the
Dang re-implementation (walk `ModuleSource.dependencies`, generate the local
ones that this SDK manages, overlay with `Workspace.withChanges`, then
generate; about twenty lines plus a fixture) is not speculative. Both
options are fully specified here; the choice is the approver's. The default
in this document is the narrowing.

### 1. Two paths, one container build

| | Released engine (`v1.0.0-beta.11`) | Engine with `#13992` + `#14038` |
|---|---|---|
| SDK entry | `initModule`, `targetRuntime`, `mod`, `modules`, `generateAll` | `detectScope`, `generateScope` (new) |
| Manifest | `[runtime] source = "python"` (v1) | `manifestVersion = 2`, `[entrypoint] kind = "dang"` |
| Module load | engine calls `runtime/` (`moduleRuntime`) | engine evaluates `sdk/entrypoint/*.dang` |
| Container build | `runtime/build.dang`, read from the runtime module's source | the same file, copied into `sdk/entrypoint/` with its externals inlined |

`runtime/main.dang` is split. The container build (`base`, `install`,
`pyConfig`, image selection, `mainObjectName`, the generated-files check)
moves to `runtime/build.dang` as `type PythonModuleBuild`, constructed from a
context `Directory`, a module subpath and a module name. Its `container`
field is the installed container: base image, `uv`, the context mounted at
`/src/<contextDir.digest>` (the module source digest served that purpose
before; a `Directory` digest keeps the uv cache keyed per source), the
`DAGGER_*` variables, and the install layers. It sets no entrypoint.

`runtime/main.dang` keeps `type PythonSdkRuntime` as a thin adapter: it
unpacks the `ModuleSource`, keeps the "no source to trust" check and the
`debug` terminal, builds `PythonModuleBuild(...).container`, and then adds
`runtime.py` at `/runtime` and sets it as the entrypoint. Those two layers
move from before the install to after it, so that the install layers are
shared with the v2 path; the runtime's behaviour is unchanged and
`e2e:runtime-call-check` proves it. `pub mainObjectName` stays as a wrapper
for `e2e:runtime-naming-check`.

The runtime module reads its own source in three places: the two image pins
in `images/*/Dockerfile` (in `build.dang`) and `runtime.py` (in `main.dang`).
The two reads in `build.dang` are fenced between `#<externals>` and
`#</externals>` markers. At generate time the authoring module splices literal
values into that block and refuses to emit a file that still mentions
`currentModule`. This is the technique of `dagger/python-sdk#24` (open, head
`c7bcd971409cbd9db54745704de35d4e266308d6`), which fences all three reads in
`runtime/main.dang` to embed the whole v1 runtime type as `[runtime] source =
"embed:runtime.dang"`. Two open series must not both add the fence: this
series carries it, in `build.dang`, and lands first; `#24` then rebases onto
it or is closed as superseded by manifest v2 for new modules. That is a
decision for the owner of both, recorded here as a recommendation to
supersede.

The entrypoint sources live in `runtime/entrypoint/`: `main.dang.tmpl` (the
template, with two placeholders) and `types.dang` (copied verbatim). They
are in a subdirectory, and the template is not a `.dang` file, because the
runtime module loads every `.dang` file in its own directory, and the
template names `ModuleEntrypoint`, which only the entrypoint driver defines.

### 2. What `generateScope` writes

For a module scope named `hello` at `.dagger/modules/hello`:

```text
.dagger/modules/hello/
├── dagger-module.toml          # v2 manifest (SDK-owned, written once)
├── pyproject.toml              # template, written once
├── src/hello/__init__.py       # template, written once
└── sdk/                        # regenerated every time
    ├── pyproject.toml, LICENSE, README.md, src/dagger/**   (as today)
    ├── src/dagger/client/gen.py                             (as today)
    └── entrypoint/
        ├── main.dang           # type Entrypoint implements ModuleEntrypoint
        ├── types.dang          # description records and the TypeDef builder
        └── build.dang          # PythonModuleBuild, externals inlined
```

`generateScope(ws, isModule, name, clients)`:

1. If `clients` is not empty, raise: client generation is
   `dagger/python-sdk#22`.
2. If `isModule` is false, return `ws` unchanged (a client-only scope has
   nothing to generate yet).
3. Normalise `ws.cwd` (strip the one leading `/` of the API form; an empty
   result is `.`) and validate it and `name` against the grammar below;
   raise otherwise.
4. If `dagger-module.toml` is absent in `ws.cwd`: render the template selected
   by the `template` setting into `ws.cwd` (merging onto existing files, as
   `initModule` does), apply the `pythonVersion`, `useUv` and `baseImage`
   settings to `pyproject.toml`, and write the v2 manifest.
5. If a manifest is present and is v1 (no `manifestVersion` key): generate
   `sdk/` as today and stop. The module keeps running on the v1 runtime,
   which the merged engine still loads. Moving it to v2 is a manual step:
   delete the manifest and regenerate.
6. Generate `sdk/` from `ws.moduleSource(".").introspectionSchemaJSON`,
   exactly as `Mod.generate` does for a `dagger-module.toml` module today,
   and write `sdk/entrypoint/main.dang` and `sdk/entrypoint/build.dang`.
7. Return the workspace. `cwd` is untouched; `dagger.toml` is never written.

The manifest is written as four lines of TOML from Dang, not through
`Query.moduleManifest`. The helper is the right tool, but a Dang module that
references it cannot load on `v1.0.0-beta.11`, which would break every
existing check. The switch is one line once this repository's minimum engine
has the helper.

**Generated literals.** The module name and the module path are the only
values written into generated Dang and TOML source. Rather than a serializer
for each language, `generateScope` accepts only values that need no escaping
in either: a name matches `^[A-Za-z0-9][A-Za-z0-9._-]*$`; a path is `.` or
one or more segments joined by `/`, each matching `^[A-Za-z0-9._-]+$` and
being neither `.` nor `..` (so the engine's default `.dagger/modules/hello`
passes). Anything else raises with the offending value. The template
placeholders (`__MODULE_NAME__`, `__MODULE_PATH__`) cannot occur in an
accepted value, so substitution cannot misfire. The e2e checks cover a name
with a quote, a backslash and a tab, and the `/`-prefixed `cwd` the engine
passes.

`detectScope(ws)` returns the directory of the nearest `pyproject.toml` at or
above `ws.cwd` (`Workspace.findUp`), `"."` for the workspace root, or `""`
when there is none.

The SDK settings become constructor fields of `PythonSdk`, next to the
existing `skipGenerateFilename`: `template` (default `"default"`; `legacy`
is rejected, see *Non-goals*), `pythonVersion`, `useUv`, `baseImage`, with
the same defaults as `initModule`'s arguments. They apply when the module is created. After that
`pyproject.toml` belongs to the user and is edited with `mod config set`; a
changed setting on an existing scope changes nothing, and the README says so.
`initModule` keeps its explicit arguments.

### 3. The generated entrypoint

The entrypoint directory holds three files. `main.dang` is assembled from
`runtime/entrypoint/main.dang.tmpl` by replacing the two placeholders with the
validated module name and path. `types.dang` is copied verbatim from
`runtime/entrypoint/types.dang`: it declares the record types of the type
description and builds `TypeDef`s from them. `build.dang` is
`runtime/build.dang` with its externals spliced. Everything that depends on
the module's shape is computed at call time by Python, so the Dang files are
the same for every module up to the two literals.

```dang
# Code generated by dagger. DO NOT EDIT.

type Entrypoint implements ModuleEntrypoint {
  let moduleName: String! = "hello"
  let modulePath: String! = ".dagger/modules/hello"

  pub types(workspace: Workspace!): [TypeDef!]! {
    let description = runtime(workspace)
      .withExec(["python", "-m", "dagger.mod", "types"], experimentalPrivilegedNesting: true)
      .file(typesOutput)
      .contents
    TypeDescriptions(text: description).typeDefs
  }

  pub call(
    workspace: Workspace!,
    receiverType: String!,
    receiverValue: JSON,
    fnName: String!,
    fnArgs: JSON!,
  ): JSON! {
    let request = JSON.encode({{
      receiverType: receiverType,
      receiverValue: receiverValue,
      fnName: fnName,
      fnArgs: fnArgs,
    }})
    let result = runtime(workspace)
      .withExec(["python", "-m", "dagger.mod", "call"], stdin: request, experimentalPrivilegedNesting: true)
      .file(callOutput)
      .contents
    (result :: JSON!)
  }

  let runtime(workspace: Workspace!): Container! {
    let root = workspace.directory("/")
    if (root.exists(join(modulePath, "pyproject.toml")) == false) {
      raise "module \"" + moduleName + "\" was generated at \"" + modulePath + "\" and is not there; run `dagger generate` after moving it"
    } else {
      let pattern = if (modulePath == ".") { "**" } else { modulePath + "/**" }
      PythonModuleBuild(
        contextDir: workspace.directory("/", include: [pattern], exclude: ["**/.venv", "**/__pycache__"]),
        subPath: modulePath,
        moduleName: moduleName,
      ).container
    }
  }

  let join(dir: String!, name: String!): String! {
    if (dir == ".") { name } else { dir + "/" + name }
  }

  let typesOutput: String! = "/dagger/types.json"
  let callOutput: String! = "/dagger/result.json"
}
```

**The type description.** `python -m dagger.mod types` writes one JSON
document: a list of type descriptions. `types.dang` decodes it with
`JSON.decode(text) :: [TypeDescription!]!` and folds each description into a
`TypeDef` with the same `withObject`/`withInterface`/`withEnum`,
`withFunction`/`withConstructor`/`withField`/`withArg`/`withEnumMember`
calls the Python builder makes today. Python decides everything (which
Python type maps to which kind, optionality, names, defaults); Dang only
transcribes. The shape, with every optional member nullable:

| Description | Members |
|---|---|
| type | `kind` (`object`, `interface`, `enum`), `name`, `description`, `deprecated`, `constructor` (function or null), `functions`, `fields`, `members` |
| function | `name` (`""` for a constructor), `description`, `deprecated`, `cachePolicy` (`DEFAULT`, `NEVER`, `PER_SESSION`, or null), `cacheTimeToLive`, `check`, `generator`, `service`, `agent`, `returns` (type reference), `args` |
| argument | `name`, `type` (type reference), `description`, `defaultValue` (JSON text or null), `defaultPath`, `defaultAddress`, `ignore` (list or null), `deprecated` |
| field | `name`, `type` (type reference), `description`, `deprecated` |
| enum member | `name`, `value`, `description`, `deprecated` |
| type reference | `kind` (`STRING_KIND`, `INTEGER_KIND`, `FLOAT_KIND`, `BOOLEAN_KIND`, `VOID_KIND`, `LIST_KIND`, `object`, `interface`, `enum`, `scalar`), `name` (for the last four), `description` (for `enum` and `scalar`, as `to_typedef` passes today), `optional`, `elem` (a type reference, for `LIST_KIND` only) |

A type reference is recursive, exactly as `to_typedef` is: a list's `elem`
is a full reference with its own `optional`, so `list[str | None]`,
`list[str] | None` and `list[list[str] | None]` are three different
descriptions, as they are three different `TypeDef`s today. `types.dang`
declares the record type with a self-reference (probe 2 verified that Dang
decodes one) and materialises it with a recursive method that applies
`withOptional(true)` first and then `withKind`, `withListOf`, `withObject`,
`withInterface`, `withEnum` or `withScalar`, in the order the Python builder
uses today, so that the two materialisations produce the same selection.
`deprecated` on a type description is transcribed for objects only; the
engine's `withInterface` and `withEnum` take none, and the Python builder
passes none. No description member carries a source map: the Python
builder emits none today, and the engine's `withSourceMap` family is the one
`with*` group the table leaves out on purpose.

Why each choice:

- **`types()` runs Python.** There is no static analyzer for Python modules;
  the type definitions come from importing the user's classes, as today. The
  exec is a plain `withExec` whose inputs are the mounted directory (the
  module directory only, without `.venv` and `__pycache__` trees), the
  vendored library and the command, so the exec's result is cached by the content of the module
  directory across sessions, which probe 1 confirmed (`CACHED` on the next
  CLI invocation). Today's type discovery runs once per session. An edit
  inside the module directory invalidates the exec; an edit elsewhere in the
  workspace does not. The Dang evaluation around the exec still reads the
  unfiltered workspace root (one `exists` probe), which is engine-side work
  per call, not a Python boot.
- **Python returns a description, Dang builds the `TypeDef`s.** Probe 1
  closed every route by which a nested process could hand the entrypoint
  `TypeDef` objects: the module-facing schema has no `load*FromID`, and
  `JSON.decode` cannot target a GraphQL object. So the entrypoint must build
  them, and the only question is who decides their content. Python already
  does (`to_typedef`, the `Module` builder); the description is that
  decision serialised, and `types.dang` is a transcription with no rules of
  its own. The Python side keeps a single decision procedure: `to_typedef`
  is rebuilt on top of the description (`to_typeref` produces the
  description, and the v1 path materialises it through the API), so the v1
  and v2 paths cannot disagree.
- **Exactly one constructor.** `Module` attaches the `""` function only to
  the object whose class is named after the module (`_module.py`,
  `is_main`), and the description carries every object, interface and enum.
  So the engine's constructor check (`validateEntrypointConstructors`) sees
  exactly one, on the class Python modules already treat as their entry
  object. The name comes from `DAGGER_MAIN_OBJECT`, which `PythonModuleBuild`
  sets from the module name with the same `mainObjectName` the v1 runtime
  uses.
- **`call()` reads a file, not stdout.** User code may `print()`. Stdout and
  stderr stay logs, attributed to the call's span by the engine; the result
  is `/dagger/result.json`, in a directory the Python process creates. A
  non-zero exit fails the exec, which fails `.file()`, which fails `call()`
  with the exec's stderr in the message. A failure is never cached.
- **The request is one JSON object on stdin.** Its `receiverValue` and
  `fnArgs` members are `JSON` scalars on the Dang side, so `JSON.encode`
  writes them as JSON text strings (probe 1 shows the exact text). Python
  decodes each of those two members exactly once. It never decodes a value
  inside `fnArgs` again: an argument whose value is the string `"null"` or
  `"{\"x\":1}"` must arrive as that string.
- **Module directory.** `workspace.cwd` is `/` for every call (see *Verified
  constraints*), so the scope path known at generate time is the only way
  to find the module, and the entrypoint raises an actionable error when it
  is not there. A module loaded from another workspace (a git ref) has no
  way to find its source at all; that is an engine gap, reported below, not
  something the entrypoint can solve.
- **Only the module directory is mounted.** The v1 runtime mounts the
  module's context narrowed by the manifest's `include` list. The v2 manifest
  has no `include`, so the entrypoint narrows to the module directory itself.
  A `pyproject.toml` that references `../shared` does not work on v2
  (*Non-goals*). The mount does not apply `.gitignore`: `sdk/` is generated
  and must be present whatever the module's ignore rules say (the `legacy`
  template ignores it), and the two trees worth dropping are named
  explicitly.

### 4. The Python side

`dagger.mod` gains a `__main__` with two subcommands. The vendored library and
the entrypoint are generated together, so the entrypoint can rely on it.

- `python -m dagger.mod types`: load the module (as `cli.load_module` does),
  build the type description of section 3, write it to
  `/dagger/types.json`. This needs no engine connection.
- `python -m dagger.mod call`: read one request object from stdin, decode
  `receiverValue` (null becomes `{}`) and `fnArgs` once each, dispatch through
  the existing `Module.get_result(parent_name, parent_state, name, inputs)`,
  and write `json.dumps(result)` to `/dagger/result.json`.

Both subcommands initialise and shut down telemetry as `app()` does, so that
spans from user code keep their parent (`TRACEPARENT` and the OTLP variables
are set on every exec). Neither connects to the engine up front:
`SharedConnection` connects on first use from the session variables the exec
provides, so a function that never touches the API runs without one, and
`types` never does.

Errors: a `ModuleError` or API error is logged the same way as today, then
the process exits with status 2 (1 for an unexpected exception). There is no
`returnError` to call from the exec (probe 1), so the message reaches the
user through the exec failure. `record_exception` is unchanged on the v1
path.

`Module` changes:

- `_converter.to_typedef(annotation)` is split into `to_typeref(annotation)
  -> dict` (the decision: kind, name, description, optionality, and the
  element reference for a list, recursively, with the existing error cases)
  and a materialisation of that dict through `dag.type_def()` in today's
  call order. `to_typedef` keeps its signature and callers, and the existing
  assertions on its selections pass unchanged.
- `_typedefs()` is split into `describe() -> list[dict]` (one description
  per object, interface and enum, per the table in section 3, built from
  the same `ObjectType`/`Function`/`Parameter` metadata) and the existing
  materialisation into `dag.module()` for `serve`/`register`, which now
  reads the description. The module description (`with_description`) has no
  home in v2 and is dropped there.
- `dispatch(request: dict) -> Any` wraps decoding and `get_result`. It is the
  unit-testable core of the `call` subcommand.

`runtime.py` and the v1 `app()`/`serve()` are untouched. `dagger/python-sdk#22`
adds a flag that the runtime entrypoint sets so that generated clients know
they run inside a module; when it lands, `python -m dagger.mod` sets it too.

### 5. What a call costs

Compared with today's runtime on the same engine:

| | today (v1) | v2 |
|---|---|---|
| module load | runtime module evaluation + one Python boot per session | Dang entrypoint evaluation + one Python boot, cached by module directory content |
| constructor / function call | one Python boot each | Dang entrypoint evaluation + one Python boot each |
| container build | once per source change | once per source change |

The per-call Python boot is unchanged. The load-time boot becomes a cache hit
whenever the module directory has not changed, which today's
`FunctionCall`-bound exec can never be. New in v2: the engine re-runs the
whole entrypoint program on every `types` and every `call`, including
`build.dang`'s `pyproject.toml` parsing and image selection. Those are pure
API calls the engine caches. Probe 1 measured the term with a handwritten
entrypoint whose execs were all cached: the engine's `loading type
definitions` span is 0.2 s and every call span rounds to 0.0 s, out of a
wall clock of 5.2–5.7 s per `dagger call` that is CLI start, session and
telemetry (the engine's own hardcoded `tiny` module took 7.3–7.7 s on the
same shared host in the same minute, which says more about the host than
about either entrypoint). The Dang evaluation is not measurable at this
resolution. The number to compare with
`hack/designs/2026-08-25-python-module-performance-ideas.md` is the one a
real module gives once patch 5 exists.

## Alternatives considered

**Static `types()`, generated at `dagger generate` time** (what the Go and
Java prototypes do, and what the spec's example shows). Rejected for Python.
Both prototypes have a static analyzer; Python's only analyzer is the
interpreter, so generation would have to build the runtime container and run
the module anyway, and every signature edit would then need `dagger generate`
before `dagger call` sees it. Dynamic `types()` keeps today's development
loop and, with content-addressed caching, is not slower when the source is
unchanged. The cost is a Python boot on first load after an edit, which the
static design pays at generate time instead.

**Return a `Module` ID and unpack it in Dang** (`loadModuleFromID(id).objects
+ .interfaces + .enums`). This is what the compatibility-bridge draft in
`#14038` proposes for legacy runtimes and would reuse `_typedefs()` verbatim.
Rejected before the probes because `Module.withObject` may rewrite a
definition before the engine's own installation does it again; rejected
after them because no ID of any kind can be loaded from Dang (probe 1,
Finding 5).

**Return `TypeDef` IDs from Python, one per line, and load them in Dang.**
The first draft of this design. Probe 1 showed it cannot work: the schema
the engine serves to modules has no `load*FromID` field, Dang cannot call
`node(id:)` with a type, and `JSON.decode` cannot target a GraphQL object.
Replaced by the description of section 3, which also removes the question of
whether an ID minted in one session loads in another.

**Return the result on stdout** (Go and Java prototypes). Python user code
prints. Rejected in favour of a result file.

**Cut over to v2 completely** (delete `runtime/`, `initModule`, `mod`,
`modules`, `generateAll`). Cleaner, and where this ends up. Rejected for this
change because the released engine cannot load a v2 module or call
`generateScope`, so the repository would be unusable and its CI red until an
engine release ships both changes. `dagger/java-sdk#19` made the same call.

**Two authoring modules** (a v2 provider module next to the beta one, so that
neither has to load on the other's engine). Rejected: the generation code
(`vendoredDir`, the code generator environment, template rendering) would be
duplicated or shared through a module dependency that itself fails to load.
Rewriting the two removed selections is smaller.

**Re-implement local-dependency staging in Dang** (walk
`ModuleSource.dependencies`, generate the local ones recursively, overlay
with `Workspace.withChanges`, then generate). About twenty lines, and it
would keep goal 3 whole. Rejected because `dagger/python-sdk#22` designs the
same walk as part of replacing dependencies with clients; landing it twice
means reconciling it twice. The narrowed behaviour is stated in section 0.

**Write the v2 manifest somewhere other than `dagger-module.toml`, or behind
a flag** (`dagger/java-sdk#19` writes `dagger-module.v2.toml`;
`dagger/go-sdk#36` keeps v2 behind an explicit `generate-v-2` function). Both
prototypes must stay loadable and generatable on the released engine, where
a v2 manifest would make the module unloadable. This design has no such
constraint: `generateScope` is only ever called by an engine that carries
`#13992`, and the released-engine tests call it directly from a check whose
only reader of the manifest is `introspectionSchemaJSON`, which accepts the
v2 keys. So `generateScope` writes the real file, and the v1 path is
untouched.

**A `manifestVersion` setting** so that an engine with `#13992` but not
`#14038` gets a v1 manifest. Rejected: a setting that applies only when the
manifest is absent is a no-op on every later generation, and that engine
combination is unsupported (*Non-goals*).

**Escape generated literals with a serializer per language** instead of
validating the input. A full serializer for Dang and for TOML basic strings
is more code than the grammar check, and the only values involved are a
module name and a path, for which the grammar is the useful contract anyway.

**Use `Query.moduleManifest` to write the manifest.** See *Proposed approach*,
section 2: it cannot be referenced by a module that must load on the released
engine.

**Embed the v1 runtime as `[runtime] source = "embed:runtime.dang"`**
(`dagger/python-sdk#24`, `dagger/dagger#14018`). A different engine mechanism
that keeps the container contract. Manifest v2 makes it unnecessary for new
modules; this design reuses its splicing technique and recommends
superseding it.

**Vendor `runtime.py` into the entrypoint** (`#24` inlines it). Unnecessary
once `dagger.mod` is runnable with `python -m`. The v1 runtime keeps
`runtime.py` because the v1 path is unchanged by this design, not for any
independence from the vendored library: the script is four lines that import
it.

## Affected components

- `future/manifest-v2-entrypoint.md` (this document)
- `python-sdk.dang`: `modules`/`mod` through `Workspace.sdk` with the cwd
  policy in Dang, and their docstrings; `detectScope`, `generateScope`,
  settings fields, the name and path grammar
- `mod.dang`: no `generateLocalDependencies`; shared generation helpers
  (`vendoredDir`, manifest text, entrypoint assembly) reachable from both
  `Mod.generate` and `generateScope`
- `docs/cwd-aware-discovery.md`: the SDK now applies the cwd policy
- `runtime/build.dang` (new): `PythonModuleBuild`, `PyConfig`, `CamelState`,
  `mainObjectName`, with the externals fence
- `runtime/main.dang`: thin `PythonSdkRuntime` adapter
- `runtime/entrypoint/main.dang.tmpl` (new): the `Entrypoint` template;
  `runtime/entrypoint/types.dang` (new): the description records and the
  `TypeDef` builder, copied verbatim into every module. A subdirectory is
  not loaded by the runtime module, which reads only its own `.dang` files.
- `sdk/src/dagger/mod/__main__.py` (new), `sdk/src/dagger/mod/cli.py`,
  `sdk/src/dagger/mod/_module.py`, `sdk/src/dagger/mod/_converter.py`;
  `sdk/tests/mod/test_dispatch.py` and `test_describe.py` (new);
  `sdk/tests/mod/test_registration.py` and `test_results.py` where they call
  `_typedefs()`
- `helpers/entrypoint-contract/` (new): a Go program that type-checks a
  generated entrypoint directory with dang v2.1.3 against an introspection
  JSON and the `ModuleEntrypoint` interface text (the shape of
  `dagger/go-sdk#36`'s helper of the same name, with the current interface)
- `dagger.json`: `include` admits `runtime/build.dang`,
  `runtime/entrypoint/**`, `runtime/images/**`; the positive entries must
  follow `"!runtime"`, because the engine applies the list in order
- `.dagger/modules/e2e/main.dang` and fixtures: new checks
- `README.md`: the two paths, the layout, the accepted differences, the
  narrowed generate behaviour

## Testing

### What can run where

| Check | Released engine (CI) | Merged dev engine (manual) |
|---|---|---|
| Python unit tests (`tests/mod`) | yes | yes |
| Existing e2e and `sdk-sdk` checks (v1 regression net) | yes | no: `sdk-sdk` drives a released CLI whose `module init` protocol `#13992` removed, and the repository's own `dagger.toml` registers no SDK in the new form |
| `generateScope` / `detectScope` called directly from e2e | yes | yes |
| `entrypointContractCheck` (type-check of the generated Dang) | yes | yes |
| `dagger module init python`, `dagger generate`, `dagger call` on a v2 module | no | yes |

This repository's CI is Dagger Cloud checks on a released engine. It cannot
load either pull request. The end-to-end proof therefore runs against the
merged engine described in *Verified constraints*, with the CLI from the same
build, and its results are recorded in *Progress* with the exact commands.
This is the evidence standard `dagger/python-sdk#24` used for its unreleased
engine dependency.

### Unit tests (`sdk/tests/mod/test_dispatch.py`)

- A request with `receiverValue: null` and `fnName: ""` calls the constructor.
- `receiverValue` and `fnArgs` arrive as JSON text strings and are decoded
  once; an argument whose value is the string `"null"` or a JSON-looking
  string is passed through as that string; an omitted argument with a default
  is not passed; a null value is passed as `None`.
- `describe()` returns one description per object, interface and enum, and
  only the main object carries a constructor (`test_describe.py`).
- For every annotation `test_registration.py` already covers, plus
  `list[str | None]`, `list[str] | None`, `list[list[str] | None]`, an enum,
  a `Scalar` subclass with a docstring, an interface and `Self`, the
  `TypeDef` materialised from `to_typeref` is the one `to_typedef` returned
  before the split: same GraphQL selection, compared through the query
  builder (`test_describe.py`).
- The `call` subcommand, run in-process on a module whose function raises,
  exits with status 2 and writes no result file. No engine is involved: the
  function never touches the API and the connection is lazy.

### e2e checks on the released engine

- `moduleDiscoveryCheck` (extended) with a new fixture where a registered
  module contains another registered module: the ordered result at the root,
  at the outer module, at the inner module, and at an unregistered directory
  inside the outer module next to the inner one, matches the rules quoted in
  *Verified constraints*.
- `generateDependencyOrderCheck`: a fixture pair `deps-modern/dep` and
  `deps-modern/app`, both `dagger-module.toml` modules, `app` depending on
  `../dep`, neither with committed bindings. Generating `app` alone fails
  with the runtime's "run `dagger generate` and commit" error; generating
  `dep`, applying it with `Workspace.withChanges`, then generating `app`
  succeeds. This pins the narrowed behaviour of section 0.
- `detectScopeCheck`: from a nested directory of a fixture with
  `pyproject.toml`, `detectScope` returns the fixture root; from a directory
  with none, `""`.
- `generateScopeInitCheck`: on an empty scope, `generateScope(isModule: true,
  name: "scope-init", clients: [])` returns a workspace whose `cwd` is
  unchanged, holding a `manifestVersion = 2` manifest with `source =
  "./sdk/entrypoint"`, the template's `pyproject.toml` and package, a
  generated `sdk/src/dagger/client/gen.py`, a `main.dang` that declares
  `implements ModuleEntrypoint` and names the module, and a `build.dang` that
  contains the image pins and no `currentModule`.
- `generateScopeRejectsUnsafeNameCheck`: names containing a double quote, a
  backslash or a tab, and a `cwd` containing a quote, each raise before any
  file is written; `generateScopeInitCheck` runs with the `/`-prefixed
  `cwd` the engine passes and a scope under `.dagger/`.
- `entrypointContractCheck`: the entrypoint directory from
  `generateScopeInitCheck` (`main.dang`, `types.dang`, `build.dang`)
  type-checks with `helpers/entrypoint-contract`
  against the introspection JSON of the runtime module's source
  (`engineVersion = "v1.0.0-0"`, so the released engine serves the view
  that has `Workspace.cwd` and the other fields the template selects) plus
  the interface text. This is the guard that CI can actually run on the file
  every v2 module ships.
- `generateScopeRegenerateCheck`: on a committed v2 fixture, the manifest and
  user files are byte-identical after regeneration; `sdk/` is refreshed.
- `generateScopeKeepsV1Check`: on a committed v1 fixture, the manifest is
  untouched and no entrypoint is written.
- `generateScopeRejectsClientsCheck`: a non-empty `clients` list raises an
  error that names the clients work.
- The existing `runtimeCallCheck`, `runtimeNamingCheck`, `initCheck`,
  `templateCheck`, `generateCheck`, `tomlGenerateCheck`, the discovery checks
  and the `sdk-sdk` suite guard the v1 path, including the rewritten
  `modules`/`mod`.

### On the merged dev engine

Probes 0 and 1 ran on 2026-09-03 with a handwritten entrypoint next to a copy
of `tiny`, in a scratch workspace, with the released CLI (`v1.0.0-beta.11`)
and the CLI built from the merge; their results are in *Verified
constraints* and summarised in *Progress*. Steps 2 to 5 run after patch 5 and
before patch 6; a failure at any step stops the plan until the design is
revised.

0. Done. `dagger call tiny hello` prints `hello`; the authoring module loads
   but cannot be called (section 0).
1. Done. `workspace.cwd` is `/` from the root and from inside the module;
   `workspace.directory("/")` is readable; a nested exec during `types` and
   `call` reaches the API; the exec is `CACHED` on the next CLI invocation;
   `currentFunctionCall` resolves in Dang and not in the exec; the request
   text is as quoted; no ID can be loaded from Dang; wall clock as in
   section 5.
2. In a scratch workspace: install this repository as SDK `python`, `dagger
   module init python --name=hello`, inspect the files against the layout in
   section 2.
3. `dagger call hello container` (default template), then a fixture that
   exercises every description member: a constructor argument, arguments
   with a default, `DefaultPath`, `Ignore` and a deprecation, a function
   returning another object of the module, an interface, a `Scalar`
   subclass, `list[str | None]` and `list[list[str] | None]`, an enum with
   member docs and a deprecated member, a deprecated field and function, a
   cache TTL, one function per `check`/`generate`/`up`/`agent` flag, and a
   function that raises. The engine's introspection of the module
   (`dagger functions` and the GraphQL `__type` of each object) is compared
   field by field with the same fixture loaded on the v1 path; this is the
   test that the transcription in `types.dang` drops nothing. Also:
   `dagger module init python --name=old --template legacy` is rejected with
   the *Non-goals* message.
4. Edit a function signature, `dagger call` again without `dagger generate`:
   the new signature is served.
5. `dagger generate` on the module: only `sdk/` changes.

## Risks

- **The engine side was unverified by anyone until probes 0 to 2.** Steps
  2 to 5 can still fail for engine reasons. When one does, the finding is
  the deliverable and the design records the workaround or the block; the
  plan does not stall on fixing the engine.
- **Two transcriptions of one decision.** `to_typeref` decides, and both
  the Python materialisation (v1) and `types.dang` (v2) transcribe. A
  member added to the description without its Dang counterpart is silently
  dropped on v2. Contained by the description table in section 3 being the
  contract, `entrypointContractCheck` type-checking `types.dang` against the
  schema, and the merged-engine step 3 comparing a fixture's `dagger
  functions` output on both paths.
- **Dang evaluation per call.** Measured inside the noise on a handwritten
  entrypoint (section 5); a real module's `build.dang` does more. Step 3
  records the number for the fixture.
- **Narrowed generate on the released engine.** Section 0. Users with an
  ungenerated local dependency generate it first; `#22` restores recursive
  generation.
- **Two generation paths.** `Mod.generate` (v1) and `generateScope` (v2)
  share `vendoredDir` and the manifest/entrypoint helpers, but they are two
  entry points until the v1 path is removed. Contained by keeping the
  module-specific logic in one place (`mod.dang`) with two thin callers.
- **Open pull requests on the same files.** `#22` touches
  `sdk/src/dagger/mod/cli.py` and `_converter.py`, which patch 4 also edits,
  and re-implements dependency staging that section 0 drops; `#24` touches
  `runtime/main.dang`, `mod.dang`, `python-sdk.dang` and `dagger.json`. This
  design is written to compose with `#22` (`clients` is the seam; the
  runtime flag is noted in section 4) and to supersede `#24`'s mechanism for
  new modules; the fence conflict is resolved by the landing order in
  section 1.
- **Schema view on the released engine.** `generateScopeInitCheck` generates
  bindings from the released engine's oldest schema view (a v2 manifest
  carries no `engineVersion`). The check asserts presence and markers, not
  binding content, so this does not make it wrong; it is why the check is not
  evidence that generated bindings match the merged engine, and why
  `entrypointContractCheck` takes its schema from a `v1.0.0-0` source.

## Findings to report

### Properties of the engine changes

Reproduced on the merged engine on 2026-09-03 (probes 0 and 1) unless
noted; to be reported to the author of `#14038` and `#13992`.

1. **A v2 module cannot be consumed from outside its own workspace.** `types`
   and `call` receive `Query.currentWorkspace`: the caller's workspace and
   `cwd`. For a module loaded from a git ref the workspace does not contain
   the module at all, and no argument tells the entrypoint where its source
   is. For a local module the SDK can bake the scope path into the
   entrypoint, which is the workaround this design uses. The spec says the
   engine "passes the same module workspace" and that the entrypoint "can
   read a file such as `go.mod` above the module directory", which needs
   either `cwd` set to the module directory (as `generateScope` gets) or a
   `Directory` argument for the module's context. Any SDK that builds from
   source hits this; only the hardcoded reference module does not.
2. **The v2 manifest has no `include` list.** An entrypoint that mounts the
   module's context cannot narrow it the way the v1 runtime did, and a
   module whose build references files above its directory has no way to
   declare them.
3. **No cache-policy signal reaches the entrypoint.** `call()` has no nonce
   or policy argument, so an entrypoint that execs a container cannot honour
   `FunctionCachePolicy.Never`/`PerSession`.
4. **No structured error channel.** `call(...): JSON!` can only raise.
   `Query.currentFunctionCall` is reachable from the Dang program but not
   from a nested exec it starts, where it fails with `get field "name":
   reflect: call of reflect.Value.Field on zero Value` instead of a clear
   error. An SDK whose implementation runs in a container loses
   `dagger.Error` values.
5. **A nested process cannot hand `TypeDef`s to the entrypoint.** The schema
   served to modules has no `load*FromID`, and Dang has no way to turn an ID
   into an object. So an entrypoint must build every `TypeDef` in Dang from
   data; the compatibility-bridge draft's `types()` ("the value is a
   `ModuleID`; load the module and return `objects`") cannot be written.
6. **No module-level description.** `types()` has no place for the module
   docstring that `Module.withDescription` carries today.

### Consequences for this repository, not engine defects

- An engine with `#13992` but not `#14038` is unsupported by this SDK
  (*Non-goals*). Not reported.
- `Query.moduleManifest` cannot be adopted by a module that must load on a
  released engine. A property of this repository's engine floor.
- Two of the three beta selections the SDK UX branch removes
  (`currentModule.asSDK`, `generateLocalDependencies`) had replacements on
  both engines, one of them narrower; the third (`initModule`/`targetRuntime`)
  is simply not called by the new engine. Worth telling the author as a data
  point on migration cost.

## Implementation plan

StGit patch series on top of `dagger/python-sdk`
`2482284f3f298adb5ac350981541129cb0860528`. Each patch carries
`Signed-off-by: Yves Brissaud <yves@dagger.io>` and no other trailer.

1. **`future: design Python modules as manifest v2 entrypoints`** — this
   document.

2. **`python-sdk: stop selecting the fields the SDK UX engine removes`**
   `python-sdk.dang`: `modules` and `managedModuleAtOrAbove` read
   `ws.sdk(name: currentModule.name).modules` and apply the cwd policy in
   Dang; docstrings and `docs/cwd-aware-discovery.md` updated. `mod.dang`:
   `Mod.generate` without `generateLocalDependencies`. e2e: the nested
   registered-module fixture for `moduleDiscoveryCheck` and
   `generateDependencyOrderCheck`. This is the patch that makes probe 0
   possible.

   Probes 0 and 1 ran before this patch was written; their outcomes are in
   *Verified constraints* and *Progress*.

3. **`runtime: move the container build into a shared type`**
   `runtime/build.dang` per section 1, with the `#<externals>` fence around
   the two image reads. `runtime/main.dang` as the thin adapter that adds
   `runtime.py` and the entrypoint after the install. Behaviour-neutral;
   `e2e:runtime-call-check` and `e2e:runtime-requires-generated-files-check`
   prove it.

4. **`sdk: describe module types as data and dispatch calls from a request`**
   `_converter.py`: `to_typeref`, with `to_typedef` materialising it.
   `_module.py`: `describe()`; `_typedefs()` materialises it; `dispatch(request)`
   decodes once and calls `get_result`. `cli.py`: `run_types()` and
   `run_call()` that load the module, run, and write the output files,
   without connecting up front and with telemetry set up as in `app()`;
   `__main__.py` maps `types`/`call` to them. `tests/mod/test_describe.py`,
   `tests/mod/test_dispatch.py`. The v1 entry (`app`) is untouched.

5. **`python-sdk: implement the module-max SDK interface`**
   `python-sdk.dang`: settings fields; `detectScope`; `generateScope` per
   section 2 with the `cwd` normalisation, the name and path grammar and the
   `legacy` template rejection. `mod.dang`: v2 manifest text;
   `entrypointDir(name, path)` assembling `main.dang` from
   `runtime/entrypoint/main.dang.tmpl`, copying `runtime/entrypoint/types.dang`,
   and splicing `build.dang` from `runtime/build.dang` with the
   `currentModule` guard. `dagger.json`: include list.
   `runtime/entrypoint/main.dang.tmpl` and `types.dang`: section 3.

6. **`e2e: cover scope detection and manifest v2 generation`**
   `helpers/entrypoint-contract/` and the `entrypointContractCheck`; fixtures
   `fixtures/scope/app` (with `pyproject.toml` and a nested directory),
   `fixtures/v2/app` (a committed v2 module that the merged engine has
   loaded) and `fixtures/v1-keep/app`; the remaining checks listed under
   *Testing*, registered in `dagger.toml` where `generateAll` must skip them.

7. **`docs: describe the manifest v2 path`**
   `README.md`: the two paths, the layout, the accepted differences, the
   narrowed generate behaviour, how to try v2 on a dev engine.

The merged-engine steps 2 to 5 of *Testing* run after patch 5 and before
patch 6, so that patch 6's fixture is a module the engine has actually
loaded.

## Progress

- Orientation: done. Repository `dagger/python-sdk`, base
  `2482284f3f298adb5ac350981541129cb0860528`. Design home `future/`. VCS:
  StGit. Host: GitHub. CI: Dagger Cloud checks from `dagger.toml`, on a
  released engine (`v1.0.0-beta.11`). Sign-off `Signed-off-by: Yves Brissaud
  <yves@dagger.io>`; no AI attribution. Engine sources read at the heads named
  above; `dagger/go-sdk#36` and `dagger/java-sdk#19` read for precedent.
- Design and plan: this document, revised twice after review.
- Plan review, round 1: two independent reviewers (a skeptic and a
  design/spec reviewer), both "rework". Shared blocker: the authoring module
  selects two fields the SDK UX branch removes and would not load on the
  merged engine; resolved by section 0. Also folded in: the entrypoint
  template must not be a `.dang` file in the runtime module; the v1 adapter
  must keep `runtime.py` and the entrypoint; the request envelope's JSON
  members are text strings (known, not an engine detail); the result
  directory must be created; the `manifestVersion` setting was a no-op and
  is gone; a static type-check of the generated entrypoint in CI;
  `workspace.directory("/")`, cross-session ID loading,
  `currentFunctionCall` reachability and the per-call Dang evaluation cost
  added to the probes; `#22`'s real overlap corrected; the landing order with
  `#24` stated; git-ref consumption made an explicit non-goal; findings split
  into engine properties and repository consequences; machine-local
  references removed.
- Plan review, round 2: the design/spec reviewer "approve with changes", the
  skeptic "rework" on one point, both reviewers verified the `Workspace.sdk`
  route on both engines. Folded in: dropping `generateLocalDependencies` is
  a real behaviour change on the released engine, so goal 3 is narrowed
  explicitly, with `generateDependencyOrderCheck` pinning it and the Dang
  re-implementation recorded as the rejected alternative that `#22` owns;
  probes 0 and 1 now run before any entrypoint code, with `CACHED` in the
  trace as the cross-session evidence; the error path was made conditional
  on the `currentFunctionCall` probe (superseded by the probe result: not
  reachable from the exec); generated literals are validated against a
  grammar instead of escaped; the merged engine is described by a
  reproducible recipe and tree hash; `entrypointContractCheck` takes a
  `v1.0.0-0` schema; the cwd policy is quoted rule for rule with a
  nested-module fixture; `#13992` alone is an explicit non-goal; the caching
  claim is scoped to the exec; `subPath` normalisation in the template
  (superseded: the cwd fallback is gone); `WorkspaceSDK.modules[].name`
  differs between the engines and is not read; telemetry setup in the
  subcommands stated.
- Merged dev engine: built from tree
  `d610a2004e4e146795ee356e602e79e2f341c276` (image plus CLI). Probes 0 and
  1 done on 2026-09-03; results in *Verified constraints*. Consequences
  folded into this revision: `types()` returns a description that Dang
  materialises (the ID route cannot work); the `cwd` fallback is gone
  (`cwd` is always `/`); the error path is settled (no `returnError` from
  the exec); the Dang evaluation cost is measured; two engine findings
  added (`load*FromID` absent from the module-facing schema; the
  `currentFunctionCall` failure mode inside a nested exec).
- Plan review, round 3: the design/spec reviewer "approve with changes",
  the skeptic "rework" on four points. Folded in: `Workspace.cwd` arrives
  in API form with a leading `/`, and the default scope lives under
  `.dagger/`, so the path grammar and a normalisation step are restated;
  the type reference is recursive with per-node optionality (probe 2
  verified self-referencing records), transcribed in today's call order;
  the mount no longer applies `.gitignore` and the `legacy` template is
  rejected on v2; scalar and enum references carry their description;
  `deprecated` and source maps stated; a comprehensive fixture compared
  field by field on both paths; the probe record below; stale sentences
  from before the probes corrected. Not folded in: the skeptic holds that
  narrowing goal 3 (section 0) violates the settled decision; that is the
  decision requested of the approver. Review rounds are capped at three.
- Approval: waiting, with one decision requested (section 0).

## Probe record

Merged engine: tree `d610a2004e4e146795ee356e602e79e2f341c276` (see
*Verified constraints*), image loaded into Docker and started as a
container without the debug port (which collided with another engine on the
host), runner host `docker-container://<that container>`; the released
`v1.0.0-beta.11` CLI for probes 0 and 1, the CLI built from the same tree
(`dagger version` reports `7ebd6da5`) to confirm `dagger module list` and
`dagger module init --help`.

Scratch workspace: a `git init`ed directory with `dagger.toml` registering
`[modules.tiny] source = "tiny"` (a copy of the engine's
`.dagger/modules/tiny`), `[modules.probe] source = "probe"`, and, for probe
0, this repository at `2482284f` extracted with `git archive` into
`python-sdk/`.

Probe 0:

```console
$ dagger call tiny hello
hello
$ dagger functions -m python-sdk          # loads: lists the seven functions
$ dagger call -m python-sdk skip-generate-filename
Error: ... field "generateLocalDependencies" not found in Dagger.ModuleSource
```

Probe 1, module `probe` with the manifest of section *Problem* and
`entrypoint/main.dang`:

```dang
type Entrypoint implements ModuleEntrypoint {
  let tool: Container! {
    container.from("alpine:3.22").withExec(["apk", "add", "--no-cache", "curl", "jq"])
  }

  let info(workspace: Workspace!): String! {
    "cwd=" + workspace.cwd + " root-entries=" + workspace.directory("/").entries.join(",")
  }

  pub types(workspace: Workspace!): [TypeDef!]! {
    let ids = tool
      .withFile("/types.sh", workspace.directory("/").file("probe/types.sh"))
      .withEnvVariable("PROBE_INFO", info(workspace) + " dang-fncall=" + (currentFunctionCall.name rescue "dang-fncall-unreachable"))
      .withExec(["sh", "/types.sh"], experimentalPrivilegedNesting: true)
      .file("/dagger/types")
      .contents
    let str = typeDef.withKind(TypeDefKind.STRING_KIND)
    [
      typeDef
        .withObject("Probe", description: DecodeProbe().recs + " " + DecodeProbe().tree)
        .withConstructor(function("", typeDef.withObject("Probe")))
        .withFunction(function("hello", str).withArg("who", str)),
    ]
  }

  pub call(workspace: Workspace!, receiverType: String!, receiverValue: JSON, fnName: String!, fnArgs: JSON!): JSON! {
    let request = JSON.encode({{ receiverType: receiverType, receiverValue: receiverValue, fnName: fnName, fnArgs: fnArgs }})
    let dangFnCall = (currentFunctionCall.name rescue "dang-fncall-unreachable")
    if (fnName == "") {
      ("{}" :: JSON!)
    } else {
      let result = tool
        .withFile("/call.sh", workspace.directory("/").file("probe/call.sh"))
        .withEnvVariable("PROBE_INFO", info(workspace) + " dang-fncall=" + dangFnCall)
        .withExec(["sh", "/call.sh"], stdin: request, experimentalPrivilegedNesting: true)
        .file("/dagger/result.json")
        .contents
      (result :: JSON!)
    }
  }
}
```

`probe/types.sh` builds a `TypeDef` through the nested session with `curl`
and `jq` (`typeDef { withObject(name: "Probe") { id } }`, `function(name:
"", returnType: <id>)`, `withConstructor`, `withFunction`, then
`currentFunctionCall { name parentName }`) and writes the resulting ID to
`/dagger/types`; `probe/call.sh` reads the request from stdin, queries
`currentFunctionCall { name parentName }`, and writes a JSON string holding
the raw request, that query's response and `PROBE_INFO` to
`/dagger/result.json`. An earlier version of `types()` returned
`JSON.decode(ids) :: [TypeDef!]!` and, before that,
`ids.split("\n").map { id => loadTypeDefFromID(id: (id :: TypeDefID!)) }`;
those are the two failures quoted in *Verified constraints*.

```console
$ dagger call probe hello --who world                      # from the root
request={"fnArgs":"{\"who\":\"world\"}","fnName":"hello","receiverType":"Probe","receiverValue":"{}"} | exec-fncall={"errors":[{"message":"get field \"name\": reflect: call of reflect.Value.Field on zero Value","path":["currentFunctionCall","name"]}],"data":null} | cwd=/ root-entries=.git/,dagger.toml,probe/,python-sdk/,tiny/ dang-fncall=hello
$ (cd probe && dagger call probe hello --who world)        # same output, cwd=/
$ dagger --progress plain -v call probe hello --who world  # second run: every withExec CACHED
$ time dagger -q call probe hello --who world              # 5.2–5.7 s wall, x2
$ time dagger -q call tiny hello                           # 7.3–7.7 s wall, x2
```

Probe 2, `probe/entrypoint/decode.dang`, loaded by the same `types()`:

```dang
type Rec { name: String! }
type Node { name: String!, optional: String, children: [Node!]! }
type DecodeProbe {
  pub recs: String! {
    let list = JSON.decode("[{\"name\":\"a\"},{\"name\":\"b\"}]") :: [Rec!]!
    "recs=" + list.map { r => r.name }.join(",")
  }
  pub tree: String! {
    let root = JSON.decode("{\"name\":\"r\",\"children\":[{\"name\":\"c\",\"children\":[]}]}") :: Node!
    "tree=" + root.name + "/" + root.children.map { c => c.name }.join(",") + " optional=" + (root.optional ?? "null")
  }
}
```

```console
$ echo '{ __type(name: "Probe") { description } }' | dagger query -m probe
{ "__type": { "description": "\nrecs=a,b tree=r/c optional=null\n" } }
```
