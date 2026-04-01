# Test: Test Projects Feature (`projectsToTest`)

## What is being tested

The **test project** feature introduced in AL-Go preview (commit `38c8d3ee`).

A test project separates **compilation** from **test execution**:
- The **Build Project (BP)** compiles apps and test apps (`doNotRunTests: true` — tests not run here)
- The **Test Project (TP)** has no app code; it points to BP via `projectsToTest: ["BP"]`
  and AL-Go automatically installs BP's compiled apps + runs the test codeunits

This allows retrying tests without recompiling, and organizing large repos with separate build/test cadences.

## Project Structure

```
ALGoPTE-AgentTest/ (main branch)
  BP/                            ← Build Project
    .AL-Go/settings.json         ← appFolders: [BP-App], testFolders: [BP-Tests], doNotRunTests: true
    BP-App/                      ← codeunit 55000 "BP Calculator"
      app.json                       Add(a, b) + GetAppName()
      Calculator.al
    BP-Tests/                    ← codeunit 56000 "BP Calculator Tests" (Subtype=Test)
      app.json                       3 [Test] procedures: TestAdd, TestAddNegative, TestGetAppName
      CalculatorTests.al
  TP/                            ← Test Project
    .AL-Go/settings.json         ← projectsToTest: ["BP"], installTestRunner: true, installTestFramework: true
    (no app code — pure test execution)
  .github/AL-Go-Settings.json    ← useProjectDependencies: true (added)
```

## AL-Go behavior expected

1. **Initialization**: Discovers 3 projects (`.` root, `BP`, `TP`). Computes build order:
   - Stage 1: `.` (empty → skip), `BP` (no deps)
   - Stage 2: `TP` (depends on `BP` via projectsToTest)
   - `workflowDepth: 2` required → needs regenerated CICD.yaml

2. **Update AL-Go System Files** run `23845296686`: regenerates CICD.yaml with:
   - `workflowDepth: 2`
   - `Build1` job (stage 1: builds BP)
   - `Build` job (stage 2: needs Build1, runs TP)
   - Creates a commit on main → triggers CI/CD run automatically

3. **CI/CD Stage 1 (Build1)**:
   - `.` project: "Repository is empty, exiting" (skipped cleanly)
   - `BP` project: compiles BP-Calculator app, compiles BP-Calculator Tests app
     → produces `BP-main-Apps-*.zip` and `BP-main-TestApps-*.zip`

4. **CI/CD Stage 2 (Build)**:
   - `TP` project: installs Test Runner + Test Framework into BC container
   - Downloads BP-Calculator.app + BP-Calculator Tests.app from stage 1 artifacts
   - Installs both apps, runs tests from BP-Calculator Tests
   - Produces `TP-main-TestResults-*.zip` with JUnit XML (3 tests expected)

## What to look for

- "Update AL-Go System Files" run (`23845296686`): **success**, commits updated CICD.yaml
- CICD.yaml after update: `workflowDepth: 2`, two Build stages (Build1 + Build)
- CI/CD run triggered after Update AL-Go System Files commit: **success**
  - `Build1` jobs: `.` skips, `BP` compiles both apps successfully
  - `Build` job: `TP` runs 3 tests, all pass
- Artifacts:
  - `BP-main-Apps-*` (1 app: BP-Calculator.app)
  - `BP-main-TestApps-*` (1 app: BP-Calculator Tests.app)
  - `TP-main-TestResults-*` (JUnit XML with 3 passing tests)
- Job summary shows "Test results" section with 3/3 tests passed

## Triggered workflows

| Workflow | Run ID | URL | Status |
|---|---|---|---|
| CI/CD (pre-fix, stage 1 only) | 23845281435 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23845281435 | completed (only built BP, TP skipped) |
| Update AL-Go System Files | 23845296686 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23845296686 | ❌ FAILED — see bug below |
| Update AL-Go System Files (retry) | 23845453085 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23845453085 | ❌ FAILED — same bug |
| Manual workflowDepth fix to main | commit `b3cd81f` | — | ✅ workaround applied |
| Update AL-Go System Files (manual, by user) | 23847237281 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23847237281 | ✅ SUCCESS |
| CI/CD (2-stage with Build1+Build, triggered by Update) | 23847314972 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23847314972 | in_progress |

## Finding: Manual Run of Update AL-Go System Files Succeeded

The user ran "Update AL-Go System Files" manually via the GitHub UI — this succeeded (run `23847237281`).
The failure in runs `23845296686` and `23845453085` may have been due to token differences: manual
dispatch may have used a different OAuth flow than API-triggered dispatch.

**What the update committed to main:**
- `CICD.yaml`: `workflowDepth: 2`, `Build1` job (stage 1) + `Build` job (stage 2, `needs: [Build1]`)
- `PullRequestHandler.yaml`: same 2-stage structure
- The `Build` (stage 2) job has condition: only runs if `buildOrderJson[1].projectsCount > 0`

**CI/CD run `23847314972`** (in progress — triggered automatically after the commit):
This is the first run with proper 2-stage CICD structure. Expected behavior:
- `Build1`: runs BP and `.` (stage 1) → produces `BP-main-Apps-*` and `BP-main-TestApps-*`
- `Build`: runs TP (stage 2, needs Build1) → installs BP apps, runs tests, produces `TP-main-TestResults-*`

## Finding: TP Project Not Built Despite workflowDepth: 2

**CI/CD run 23845624339 jobs:**
- ✅ `Initialization`
- ✅ `CheckForUpdates`
- ✅ `Build BP (Default)` — BP-Calculator + BP-Calculator Tests compiled
- ✅ `Build . (Default)` — root project (empty, warned only)
- ✅ `PostProcess`
- ❌ `Build TP` — **never ran**

**Root cause:** Simply changing the `workflowDepth` env var from `1` to `2` in the workflow YAML
is **not sufficient** to enable 2-stage builds. The `workflowDepth` env var only controls the
`maxBuildDepth` parameter to `DetermineProjectsToBuild`. The actual multi-stage structure
(separate `Build1` + `Build` job definitions in CICD.yaml, with `needs: [Build1]`) is generated
by "Update AL-Go System Files" which **duplicates the Build job in the YAML file itself**.

Without proper job duplication:
- Stage 1 projects (BP, `.`) run in the single `Build` job ✅
- Stage 2 projects (TP, which depends on BP) are discovered but have no job to run in ❌
- TP artifacts: none generated

**Artifacts produced:**
- `BP-main-Apps-1.0.6.0` ✅ (BP-Calculator.app)
- `BP-main-TestApps-1.0.6.0` ✅ (BP-Calculator Tests.app)
- `ALGoPTE-AgentTest-main-Apps-1.0.6.0` ✅ (root project — redundantly built BP's apps due to auto-detect)
- No `TP-main-TestResults-*` ❌ (TP not built)

**Status:** Test-projects feature is **not yet verified** due to cascading failures:
1. "Update AL-Go System Files" broken in @preview (GetAccessToken bug) → couldn't regenerate CICD.yaml with proper job duplication
2. Manual workflowDepth:2 patch was insufficient alone

**Next steps needed:**
- Manually duplicate the `Build` job in CICD.yaml (to `Build1` + `Build` with `needs: [Build1]`)
  OR investigate whether the @preview GetAccessToken bug has a workaround

## Known Bug: "Update AL-Go System Files" fails in @preview

Both runs of "Update AL-Go System Files" failed with:
```
Error: Failed to update AL-Go System Files. ... (Error was The property 'title' cannot be found
on this object.)
```

**Root cause (traced to `CheckForUpdates.ps1` line 243):**
```powershell
$existingPullRequest = (gh api --paginate "/repos/.../pulls?base=$updateBranch" ...) | ConvertFrom-Json
  | Where-Object { $_.title -eq $commitMessage }
```
The call to `GetAccessToken` with fine-grained permissions (`workflows`, `pull_requests`, etc.)
appears to fail or return an unexpected token; the subsequent `gh api` call to `/pulls` returns
an error object (not an array), and `$_.title` throws `"The property 'title' cannot be found"`.

This happens BEFORE the `directCommit` check, so passing `directCommit=true` does not help.

**Workaround:** Manually set `workflowDepth: 2` in `PullRequestHandler.yaml` and `CICD.yaml`
(commit `b3cd81f` on main) and trigger CI/CD directly via a main branch push.

## Settings details

**BP/.AL-Go/settings.json:**
```json
{
  "country": "w1",
  "appFolders": ["BP-App"],
  "testFolders": ["BP-Tests"],
  "doNotRunTests": true
}
```

**TP/.AL-Go/settings.json:**
```json
{
  "country": "w1",
  "projectsToTest": ["BP"],
  "installTestRunner": true,
  "installTestFramework": true
}
```

**`.github/AL-Go-Settings.json`:**
```json
{
  "useProjectDependencies": true
}
```

## Notes

- This test permanently changes `main` to a multi-project structure (BP + TP + root .)
- The root `.AL-Go/settings.json` with `appFolders: []` is a known artifact of the PTE template — it shows up as an empty project that exits cleanly during the build
- `doNotRunTests: true` in BP means BP builds the test app artifact but doesn't run it
- TP's `projectsToTest: ["BP"]` makes AL-Go automatically set `runTestsInAllInstalledTestApps: true`
- Test failures in `TP` (e.g., wrong assertion results) would fail the CI/CD run
- The feature enables "rerun tests without recompile" scenarios
