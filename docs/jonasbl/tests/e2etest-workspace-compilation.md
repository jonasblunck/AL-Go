# Test: Workspace Compilation — 8-App Diamond Dependency Graph

## What is being tested

The `workspaceCompilation` feature introduced in AL-Go preview (requires BC v28 ALTool).

Instead of compiling apps one-by-one in dependency order (sequential), the ALTool's
`workspace compile` command receives all app folders at once, computes the full dependency
graph internally, and compiles apps **in parallel** wherever the graph allows.

Key settings under test:
- `workspaceCompilation.enabled: true` — activates the new compilation path
- `workspaceCompilation.parallelism: -1` — use all available CPUs
- `doNotPublishApps: true` — compile-only, no BC container needed (fast)

The compilation pipeline is:
1. `CompileApps` action creates a compiler folder (BC artifacts)
2. `Build-AppsInWorkspace` calls `altool workspace create <folders>` to generate a workspace file
3. Then calls `altool workspace compile <workspace.json>` with `--maxcpucount`
4. The ALTool resolves the dependency graph and compiles in parallel layers
5. `RunPipeline` exits early (doNotPublishApps=true, no container)

## Dependency graph designed to maximise parallelism

```
Layer 0 (can compile in parallel — no deps):
  WS-Foundation (codeunit 50001)
  WS-Utils      (codeunit 50101)

Layer 1 (can compile in parallel — wait for layer 0):
  WS-CoreA  (codeunit 50201) ← Foundation
  WS-CoreB  (codeunit 50301) ← Foundation + Utils

Layer 2 (can compile in parallel — wait for layer 1):
  WS-ServiceA  (codeunit 50401) ← CoreA
  WS-ServiceB  (codeunit 50501) ← CoreA + CoreB   (diamond: both CoreA and CoreB)
  WS-ServiceC  (codeunit 50601) ← CoreB

Layer 3 (must be last — waits for all services):
  WS-Integration  (codeunit 50701) ← ServiceA + ServiceB + ServiceC
```

Each codeunit calls `GetName()` from its dependencies, so the compiler must actually
resolve the symbols (not just metadata) to compile successfully.

## What to look for

- Pull Request Build conclusion: **success**
- All 8 apps compiled: `*-Apps-*.zip` artifact containing 8 `.app` files
- App names in artifact: `WS-Foundation`, `WS-Utils`, `WS-CoreA`, `WS-CoreB`,
  `WS-ServiceA`, `WS-ServiceB`, `WS-ServiceC`, `WS-Integration`
- No `*-TestApps-*` artifact (no test apps)
- `CompileApps` step logs showing `altool workspace create` and `altool workspace compile`
- Ideally: log evidence of parallel compilation (multiple compile processes)
- No compilation errors — particularly for the diamond dependency (ServiceB←CoreA+CoreB)
  and the cross-cutting Integration layer

## Test branch

`private/jonasbl/test-workspace-compilation-20260401-112937` in `jonasblunck/ALGoPTE-AgentTest`

## Pull Request

https://github.com/jonasblunck/ALGoPTE-AgentTest/pull/2

## Triggered workflows

| Workflow | Run ID | URL | Status at trigger time |
|---|---|---|---|
| Pull Request Build | 23842022000 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23842022000 | in_progress |

## App structure pushed

| App | Codeunit ID | ID Range | Dependencies |
|---|---|---|---|
| WS-Foundation | 50001 | 50000-50099 | none |
| WS-Utils | 50101 | 50100-50199 | none |
| WS-CoreA | 50201 | 50200-50299 | Foundation |
| WS-CoreB | 50301 | 50300-50399 | Foundation, Utils |
| WS-ServiceA | 50401 | 50400-50499 | CoreA |
| WS-ServiceB | 50501 | 50500-50599 | CoreA, CoreB |
| WS-ServiceC | 50601 | 50600-50699 | CoreB |
| WS-Integration | 50701 | 50700-50799 | ServiceA, ServiceB, ServiceC |

## Notes

- `doNotPublishApps: true` makes this compile-only (no BC container), so it should be fast.
- Workspace compilation only works on Windows runners — confirmed default runner is `windows-latest`.
- This is the first test of the **preview** AL-Go version (repo updated from @main to @preview).
- The new `CompileApps` action (`Actions/CompileApps/Compile.ps1`) replaces the old compilation
  path inside `RunPipeline`. Both workspace and non-workspace compilation now go through it.
- `MyApp` and `MyApp.Test` from the basic-build test are NOT in this branch's settings
  (settings.json explicitly lists only the WS-* folders), so they won't interfere.
