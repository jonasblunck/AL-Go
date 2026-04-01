# Test: Incremental Builds — modifiedApps Mode

**Test ID:** incremental-builds-20260401-120821  
**Branch:** `private/jonasbl/test-incremental-builds-20260401-120821` → PR #7  
**Repo:** `jonasblunck/ALGoPTE-AgentTest`  
**Purpose:** Verify that `incrementalBuilds.onPull_Request: true` with `mode: modifiedApps` (default)
correctly reuses baseline artifacts for unmodified apps and only recompiles the changed app.

---

## Feature Under Test

`incrementalBuilds` setting (in `.github/AL-Go-Settings.json`):
```json
{
  "incrementalBuilds": {
    "onPush": false,
    "onPull_Request": true,
    "onSchedule": false,
    "retentionDays": 30,
    "mode": "modifiedApps"
  }
}
```

**modifiedApps mode behavior:**
- For each app folder, determine if any source file changed
- If changed: recompile the app (and all apps that transitively depend on it)
- If not changed AND not in the dependency tree of a changed app: **download from baseline CI/CD run**
- `fullBuildPatterns` (default: `.github/*.json`): if any matching file is modified → full build
- `.AL-Go/settings.json` change → rebuild all apps in that project

---

## Project Structure

```
INC/
  .AL-Go/settings.json  (appFolders: [INC-Base, INC-Mid, INC-Top])
  INC-Base/             codeunit 59000 "INC Base Logic" — independent
  INC-Mid/              codeunit 59100 "INC Mid Processor" — depends on INC-Base
  INC-Top/              codeunit 59200 "INC Top App" — depends on INC-Mid
```

Dependency chain: `INC-Base ← INC-Mid ← INC-Top`

---

## PR Change

**Only `INC/INC-Top/Top.al` was modified** — one comment line + output string change.  
No `.github/*.json` or `.AL-Go/settings.json` files were touched.

---

## Expected Behavior

| App | Modified? | Expected Build Action |
|-----|-----------|----------------------|
| `INC-Base` | No | ⬇️ Downloaded from CI/CD baseline |
| `INC-Mid` | No | ⬇️ Downloaded from CI/CD baseline |
| `INC-Top` | **Yes** | 🔨 Recompiled |

**Key log messages to look for:**
- `Downloading INC-Base ... from baseline run` (or similar)
- `Downloading INC-Mid ... from baseline run`
- Compiler output only for `INC-Top`
- `Incremental build: reusing X apps from baseline`

**Overall build: PASS**

---

## Important Note on Baseline Requirement

Incremental build requires a **prior successful CI/CD run** on `main` that includes the INC project.  
The INC project was added to main in commit `9077652`.

- **Run 1 (no baseline):** Full build — all 3 INC apps compile. This is correct; AL-Go falls back to
  full build when no baseline exists.
- **Run 2 (after CI/CD baseline):** Incremental — only INC-Top recompiles. This is the interesting run.

The CI/CD baseline run is `23847786147` (triggered by main commit `9077652`).

---

## Runs

| Run | Trigger | Expected | Actual | Notes |
|-----|---------|----------|--------|-------|
| CI/CD baseline | push to main (commit 9077652) | Full build, all 3 INC apps compile | `23847786147` in_progress | Creates baseline |
| PR Build (Run 1) | PR #7 created | Full build (no INC baseline yet) | `23847840699` in_progress | Should be full build |
| PR Build (Run 2) | Re-trigger after CI/CD baseline | Incremental: only INC-Top | TBD | Re-trigger after 23847786147 completes |
