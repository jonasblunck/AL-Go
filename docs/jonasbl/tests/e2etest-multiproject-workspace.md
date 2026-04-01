# Test: Multi-Project Repository with Workspace Compilation

## What is being tested

A multi-project repository where **each project independently uses workspace compilation**.
This tests AL-Go's project discovery combined with the workspace compilation feature.

- AL-Go discovers multiple projects by scanning for subfolders containing `.AL-Go/settings.json`
- Each project runs its own workspace compilation (ALTool creates a workspace per project)
- Projects are independent (no cross-project dependencies) — both build in parallel in the same CI/CD stage
- CI/CD workflow triggers automatically on `feature/*` branch pushes

### Project structure

```
ALGoPTE-AgentTest/
  P1/                          ← Project 1: Platform layer
    .AL-Go/settings.json
    P1-Base/                   ← foundation codeunit (53000)
    P1-Helpers/                ← depends on P1-Base (53100)
  P2/                          ← Project 2: Services layer
    .AL-Go/settings.json
    P2-Core/                   ← core services codeunit (54000)
    P2-Services/               ← depends on P2-Core (54100)
```

Key settings **per project** (in each `project/.AL-Go/settings.json`):
- `artifact: "////nextmajor"` — required for ALTool workspace support
- `workspaceCompilation: { enabled: true, parallelism: -1 }` — workspace mode
- `doNotPublishApps: true` — compile-only (fast, no BC container)

### Why `feature/*` branch (not a PR)?

CI/CD workflow triggers on `feature/*` and `release/*` pushes, while PR Build is for PRs to main.
This test uses CI/CD directly to validate multi-project discovery and build.

### Cross-project dependency note

The two projects are **independent** (no cross-project dependencies) so `workflowDepth: 1`
is sufficient and both projects build in the same stage (parallel matrix jobs).

For cross-project dependency + workspace (P2 depends on P1's compiled apps), you would need:
1. `useProjectDependencies: true` in repo settings
2. "Update AL-Go System Files" to regenerate CICD.yaml with `workflowDepth: 2`
3. This is tracked as a future test scenario

## What to look for

- CI/CD conclusion: **success**
- **Two build jobs** in the matrix (one per project):
  - `Build P1 (Default)` — compiles P1-Base and P1-Helpers via workspace
  - `Build P2 (Default)` — compiles P2-Core and P2-Services via workspace
- Artifacts:
  - `P1-feature-ws-multiproject-*-Apps-*.zip` containing 2 `.app` files
  - `P2-feature-ws-multiproject-*-Apps-*.zip` containing 2 `.app` files
- Both jobs run in parallel (same build stage)
- `altool workspace create` + `altool workspace compile` appear in each project's build logs
- No `*-TestApps-*` artifacts (no test apps in either project)

## Test branch

`feature/ws-multiproject-20260401-124231` in `jonasblunck/ALGoPTE-AgentTest`

(No PR — CI/CD triggered directly by push to `feature/*` branch)

## Triggered workflows

| Workflow | Run ID | URL | Status |
|---|---|---|---|
| CI/CD | 23844702616 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23844702616 | in_progress |

## App structure

### P1 project (`P1/.AL-Go/settings.json`)

| App | Codeunit | Depends on | Key method |
|-----|---------|------------|------------|
| P1-Base | 53000 | — | `GetPlatformName()`, `GetVersion()` |
| P1-Helpers | 53100 | P1-Base | `GetFullIdentifier()` → calls Base |

### P2 project (`P2/.AL-Go/settings.json`)

| App | Codeunit | Depends on | Key method |
|-----|---------|------------|------------|
| P2-Core | 54000 | — | `GetServiceName()`, `IsEnabled()` |
| P2-Services | 54100 | P2-Core | `GetServiceStatus()` → calls Core |

## Notes

- Root `.AL-Go/settings.json` was **removed** from this branch — in a multi-project repo,
  the root `.AL-Go/settings.json` would be mistaken for a 3rd project with empty `appFolders`.
  Repo-level settings belong in `.github/AL-Go-Settings.json`.
- `application: "24.0.0.0"` used in app.json — compatible with nextmajor artifact download.
- The two workspace compilations are completely independent — each runs ALTool separately.
