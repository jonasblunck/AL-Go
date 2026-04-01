# AL-Go E2E Test Agent Instructions

## Purpose

This document tells the agent how to push AL PTE test apps to GitHub, trigger AL-Go pipelines,
read logs and artifacts, and verify functionality — all against a single dedicated test repository.

---

## Environment

| Item | Value |
|---|---|
| Test repository | `jonasblunck/ALGoPTE-AgentTest` (private, github.com) |
| AL-Go repo (this repo) | `jonasblunck/AL-Go` — where test docs are stored |
| AL-Go template | `https://github.com/microsoft/AL-Go-PTE@main` |
| Repo type | PTE (Per-Tenant Extension) |
| Default branch | `main` — **never push directly to main** |
| GitHub auth | `gh --hostname github.com` (logged in as `jonasblunck`) |
| AL-Go system files | Already installed (v8.3 workflows, 20 active) |

### Authentication

All `gh` and `git` operations targeting github.com must use:
```bash
gh --hostname github.com ...
```

To get a token for git push auth:
```bash
TOKEN=$(gh --hostname github.com auth token)
git remote set-url origin "https://jonasblunck:${TOKEN}@github.com/jonasblunck/ALGoPTE-AgentTest.git"
```

---

## Branch Strategy

Each test scenario gets its **own permanent branch** in `ALGoPTE-AgentTest` so runs and
results can be revisited at any time. Branches are **never deleted** after testing.

| Branch name pattern | Purpose |
|---|---|
| `private/jonasbl/test-<scenario>-<YYYYMMDD-HHMMSS>` | One branch per test run |

The **Pull Request Build** workflow (`PullRequestHandler.yaml`) is the primary validation
workflow. It triggers automatically when a PR is opened targeting `main`. It covers
compilation, publishing, and test execution.

> **Do NOT wait for workflows to complete.** Fire and forget: push the branch, open the PR,
> record the run ID, then move on. Multiple tests can run in parallel this way.

---

## Repo Setup & Clone

```bash
# Clone the repo (do this once per terminal session)
cd /tmp
gh --hostname github.com repo clone jonasblunck/ALGoPTE-AgentTest algotest
cd algotest

# Set push URL with token
TOKEN=$(gh --hostname github.com auth token)
git remote set-url origin "https://jonasblunck:${TOKEN}@github.com/jonasblunck/ALGoPTE-AgentTest.git"

git config user.email "jonas.blunck@example.com"
git config user.name "jonasblunck"
```

---

## AL App File Templates

### app.json (main app)
```json
{
  "id": "<new GUID>",
  "name": "<AppName>",
  "publisher": "Agent Test Publisher",
  "version": "1.0.0.0",
  "brief": "",
  "description": "",
  "privacyStatement": "",
  "EULA": "",
  "help": "",
  "url": "",
  "logo": "",
  "dependencies": [],
  "screenshots": [],
  "platform": "1.0.0.0",
  "application": "24.0.0.0",
  "idRanges": [{ "from": 50000, "to": 51000 }],
  "resourceExposurePolicy": {
    "allowDebugging": true,
    "allowDownloadingSource": false,
    "includeSourceInSymbolFile": false
  }
}
```

### HelloWorld.al (simple app — triggers a message on Customer List)
```al
pageextension 50000 "CustListExt<AppName>" extends "Customer List"
{
    trigger OnOpenPage();
    begin
        Message('App published: Hello <AppName>!');
    end;
}
```

### app.json (test app — depends on main app)
Same as above but:
- Different GUID
- `"name": "<AppName>.Test"`
- `"idRanges": [{ "from": 60000, "to": 61000 }]`
- `"dependencies": [{ "id": "<main app GUID>", "name": "<AppName>", "publisher": "Agent Test Publisher", "version": "1.0.0.0" }]`

### HelloWorld.Test.al (test codeunit)
```al
codeunit 60000 "<AppName> Test"
{
    Subtype = Test;

    [Test]
    [HandlerFunctions('HelloWorldMessageHandler')]
    procedure TestHelloWorldMessage()
    var
        CustList: TestPage "Customer List";
    begin
        CustList.OpenView();
        CustList.Close();
        if (not MessageDisplayed) then
            Error('Message was not displayed!');
    end;

    [MessageHandler]
    procedure HelloWorldMessageHandler(Message: Text[1024])
    begin
        MessageDisplayed := MessageDisplayed or (Message = 'App published: Hello <AppName>!');
    end;

    var
        MessageDisplayed: Boolean;
}
```

---

## Settings Files

### .github/AL-Go-Settings.json (repo-level)
The file already exists. Modify it to add/override repo-level settings.
Current content:
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/AL-Go-Actions/v8.3/.Modules/settings.schema.json",
  "type": "PTE",
  "templateUrl": "https://github.com/microsoft/AL-Go-PTE@main"
}
```

### .AL-Go/settings.json (project-level — single project)
```json
{
  "country": "w1",
  "appFolders": ["<AppFolder>"],
  "testFolders": ["<TestAppFolder>"]
}
```

> **Note:** `country: "w1"` is used in tests (W1 = worldwide/generic). Use `"us"` for US-specific features.
> When `testFolders` is set and `doNotPublishApps` is NOT true, AL-Go will run a full BC container
> pipeline (compile + publish + run tests). This is slower but required for actual test execution.

### To enable test execution (required for test runs):
The repo default has `doNotPublishApps` unset (defaults to false), so tests **will** run
as long as `testFolders` is populated. Do NOT add `"useCompilerFolder": true` or
`"doNotPublishApps": true` when you want test runs.

---

## Key Settings Reference

| Setting | Location | Purpose |
|---|---|---|
| `country` | project | BC localization (use `"w1"` for generic) |
| `appFolders` | project | App folders to compile |
| `testFolders` | project | Test app folders; triggers test runs |
| `doNotPublishApps` | repo or project | `true` = compiler-folder only, no tests |
| `useCompilerFolder` | repo or project | `true` = no Docker container (faster, no tests) |
| `incrementalBuilds` | repo | `{ "onPush": true, "mode": "modifiedApps" }` |
| `workspaceCompilation` | repo | `{ "enabled": true, "parallelism": -1 }` |
| `useProjectDependencies` | repo | Enable multi-project dependency graph |
| `postponeProjectInBuildOrder` | project | Defer project to last build stage |
| `projectsToTest` | project | Mark as test-only project (no build code) |
| `buildModes` | project | e.g. `["Default", "Clean"]` |
| `githubRunner` | repo | `"windows-latest"` or `"ubuntu-latest"` |
| `conditionalSettings` | repo | Context-specific overrides (by workflow, branch) |

---

## Workflow Operations

### Trigger a workflow manually (workflow_dispatch)
```bash
gh --hostname github.com api \
  --method POST \
  /repos/jonasblunck/ALGoPTE-AgentTest/actions/workflows/<workflow-file>.yaml/dispatches \
  -f ref="refs/heads/<branch>" \
  -f inputs='{"directCommit":"true"}'
```

Common workflow file names:
- `CICD.yaml` — main build
- `UpdateAlGoSystemFiles.yaml` — update AL-Go system files
- `CreateApp.yaml` — scaffold new app
- `CreateTestApp.yaml` — scaffold new test app
- `IncrementVersionNumber.yaml` — bump versions

### Wait for a workflow run to complete
```bash
# List latest runs
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs \
  --jq '.workflow_runs[:5] | .[] | "\(.id) \(.name) \(.status) \(.conclusion) \(.head_branch)"'

# Poll a specific run until done
RUN_ID=<id>
while true; do
  STATUS=$(gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID \
    --jq '.status')
  echo "Status: $STATUS"
  if [ "$STATUS" = "completed" ]; then break; fi
  sleep 30
done

# Check conclusion
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID \
  --jq '{conclusion: .conclusion, url: .html_url}'
```

### List and download artifacts
```bash
# List artifacts for a run
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID/artifacts \
  --jq '.artifacts[] | "\(.name) (\(.size_in_bytes) bytes)"'

# Download artifacts
gh --hostname github.com run download $RUN_ID --repo jonasblunck/ALGoPTE-AgentTest --dir /tmp/artifacts/
```

### Read workflow run logs
```bash
gh --hostname github.com run view $RUN_ID --repo jonasblunck/ALGoPTE-AgentTest --log 2>&1 | head -200
# Or for a specific job:
gh --hostname github.com run view $RUN_ID --repo jonasblunck/ALGoPTE-AgentTest --log-failed
```

### Get jobs summary for a run
```bash
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID/jobs \
  --jq '.jobs[] | "\(.name): \(.conclusion)"'
```

---

## Standard Test Procedure

For every test scenario, follow these steps:

### 1. Prepare
```bash
cd /tmp
rm -rf algotest
GH_HOST=github.com gh repo clone jonasblunck/ALGoPTE-AgentTest algotest
cd algotest
TOKEN=$(gh --hostname github.com auth token)
git remote set-url origin "https://jonasblunck:${TOKEN}@github.com/jonasblunck/ALGoPTE-AgentTest.git"
git config user.email "copilot-agent@github.com"
git config user.name "Copilot Agent"
```

### 2. Create a test branch
```bash
BRANCH="feature/test-<scenario>-$(date +%Y%m%d-%H%M%S)"
git checkout -b "$BRANCH"
```

### 3. Add app code + settings
Create folders, write app.json, .al files, update .AL-Go/settings.json.

### 4. Commit and push
```bash
git add -A
git commit -m "Test: <scenario description>"
git push origin "$BRANCH"
```

### 5. Wait for CI/CD to trigger, then wait for completion
```bash
sleep 30  # let GitHub register the push

RUN_ID=$(gh --hostname github.com api \
  "/repos/jonasblunck/ALGoPTE-AgentTest/actions/runs?branch=${BRANCH}&per_page=5" \
  --jq '.workflow_runs[] | select(.name == " CI/CD") | .id' | head -1)

echo "Watching run: https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID"

while true; do
  STATUS=$(gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID --jq '.status')
  if [ "$STATUS" = "completed" ]; then break; fi
  echo "  ... $STATUS"
  sleep 30
done

CONCLUSION=$(gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID --jq '.conclusion')
echo "Result: $CONCLUSION"
```

### 6. Verify results
```bash
# Check artifacts
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID/artifacts \
  --jq '.artifacts[].name'

# Check jobs
gh --hostname github.com api /repos/jonasblunck/ALGoPTE-AgentTest/actions/runs/$RUN_ID/jobs \
  --jq '.jobs[] | "\(.name): \(.conclusion)"'

# Download and inspect if needed
gh --hostname github.com run download $RUN_ID --repo jonasblunck/ALGoPTE-AgentTest --dir /tmp/artifacts/
ls /tmp/artifacts/
```

### 7. Cleanup
```bash
git checkout main
gh --hostname github.com api --method DELETE \
  "/repos/jonasblunck/ALGoPTE-AgentTest/git/refs/heads/${BRANCH}"
rm -rf /tmp/artifacts/
```

---

## Test Scenarios

### Scenario: Basic Build & Test
**Goal:** Verify a simple app compiles and tests run.

**Setup:**
- 1 main app (`MyApp/`) with a page extension
- 1 test app (`MyApp.Test/`) with a test codeunit
- `.AL-Go/settings.json`: `country: "w1"`, `appFolders: ["MyApp"]`, `testFolders: ["MyApp.Test"]`
- Do NOT set `doNotPublishApps` or `useCompilerFolder`

**Verify:**
- CI/CD conclusion: `success`
- Artifacts include `*-Apps-*.zip` and `*-TestApps-*.zip`
- Jobs include a `Run tests` step with passing tests
- Workflow summary shows "X AL tests"

---

### Scenario: Incremental Builds
**Goal:** Verify only modified apps (and their dependants) are rebuilt.

**Setup:**
- 4 apps: `app1` (base) → `app2` → `app3`; `app4` (standalone, no dependencies)
- `.github/AL-Go-Settings.json`: add `"incrementalBuilds": { "onPush": true }`, `"useCompilerFolder": true`, `"doNotPublishApps": true`, `"useProjectDependencies": true`
- `.AL-Go/settings.json`: `country: "w1"`

**Test sequence (multiple commits/runs):**
1. Initial push — all 4 apps build. Note version numbers (should all be e.g. `1.0.2.0`).
2. Modify `app1` (e.g. add `!` to message) — push. Expect: `app1`, `app2`, `app3` rebuild (new version); `app4` reuses old artifact.
3. Modify `app4` — push. Expect: only `app4` rebuilds; `app1`/`app2`/`app3` reuse prior artifacts.

**Verify** (artifact names encode version numbers):
```bash
gh --hostname github.com run download $RUN_ID --repo jonasblunck/ALGoPTE-AgentTest --dir /tmp/artifacts/
ls /tmp/artifacts/*/  # check .app filenames for version numbers
```

Artifact filenames format: `<project>-<branch>-Apps-<version>.zip` containing `<publisher>_<name>_<version>.app`.

---

### Scenario: Workspace Compilation
**Goal:** Verify parallel workspace compilation works with cross-project dependencies.

**Setup:**
- 2 projects: `P1` (app1 base, app2 depends on app1) and `P2` (app3 depends on P1/app1)
- `.github/AL-Go-Settings.json`: add:
  ```json
  "workspaceCompilation": { "enabled": true, "parallelism": -1 },
  "useProjectDependencies": true,
  "doNotPublishApps": true,
  "useCompilerFolder": true
  ```
- `P1/.AL-Go/settings.json`: `country: "w1"`
- `P2/.AL-Go/settings.json`: `country: "w1"`

**Note:** Workspace compilation currently only works on Windows runners. Ensure `githubRunner` is not set to `ubuntu-latest`.

**Verify:**
- CI/CD succeeds
- P1 and P2 each have artifact zip files
- Logs show parallel compilation in RunPipeline step

---

### Scenario: Build Modes
**Goal:** Verify multiple build modes produce separate artifact sets.

**Setup:**
- 1 app with `buildModes: ["Default", "Clean"]` in `.AL-Go/settings.json`
- `doNotPublishApps: true`, `useCompilerFolder: true`

**Verify:**
- Artifacts include both `*-Apps-*.zip` (Default) and `*-Clean-Apps-*.zip` (Clean mode)

---

### Scenario: Postpone Project in Build Order
**Goal:** Verify a test project is deferred to the last build stage.

**Setup:**
- 2 projects: `Build` (a normal app project) and `Tests` (a test-only project with `projectsToTest: ["Build"]`)
- `Tests/.AL-Go/settings.json`:
  ```json
  { "projectsToTest": ["Build"], "postponeProjectInBuildOrder": true }
  ```
- `useProjectDependencies: true` in repo settings

**Verify:**
- CI/CD logs show `Tests` project starts after `Build` completes
- Artifact for `Build` is present; no app artifacts for `Tests` (it's test-only)

---

## Troubleshooting

### "No runs found on branch"
Push triggers are filtered by paths. Changes only to `.md` files won't trigger CI/CD.
Always modify at least one `.al` or `.json` (settings) file.

### Workflow not finding apps
AL-Go auto-discovers apps by scanning up to 2 levels for folders containing `app.json`.
If `appFolders` is explicitly set in settings.json, only those exact folder paths are used.
Leave `appFolders: []` to use auto-discovery, or specify exact folder names.

### Test run very slow / timing out
Running AL tests requires a full BC Docker container on the GitHub-hosted runner.
This can take 15–30 minutes. This is expected. Use `doNotPublishApps: true` +
`useCompilerFolder: true` for compile-only checks (no test execution).

### Run shows "failure" but tests are fine
Check the jobs list — a failure in the `Update AL-Go System Files` job is non-critical.
Focus on the `Build <project>` jobs and the `Run tests` step.

### Token expired mid-test
```bash
TOKEN=$(gh --hostname github.com auth token)
git remote set-url origin "https://jonasblunck:${TOKEN}@github.com/jonasblunck/ALGoPTE-AgentTest.git"
```

---

## Important AL-Go Quirks

1. **`Update AL-Go System Files` run**: After changes to `.github/AL-Go-Settings.json` that affect
   workflow structure (e.g. `useProjectDependencies`, `buildModes`), you may need to run this
   workflow first. For simple setting changes (e.g. `incrementalBuilds`), it's not required.

2. **Version numbering**: AL-Go uses `useApproximateVersion` logic. The build number in artifact
   names (`1.0.X.0`) increments with each run. The run number is embedded in the patch version.
   Don't assume a specific number — assert patterns like `*_app1_1.0.*.app` or compare between runs.

3. **Incremental builds reuse**: When incremental builds is on, unchanged apps are copied from the
   latest artifact of the previous successful run on the same branch. The artifact version of reused
   apps will be from an earlier build (lower patch number).

4. **Multi-project repos**: When `useProjectDependencies: true`, the number of build stages changes
   based on dependency depth. Running `Update AL-Go System Files` regenerates the CICD.yaml with
   the correct number of parallel build jobs. This may need to be done before the first multi-project CI/CD run.

5. **CI/CD workflow name**: The workflow is named `" CI/CD"` (with a leading space). When filtering
   runs by name, use `select(.name == " CI/CD")`.

6. **Artifact naming**: Format is `<project>-<branch>-Apps-<repoVersion>.<buildNumber>.<revision>`.
   For a single-project repo (no `projects` in root), the project name is the repo name or `.`.
   For multi-project repos, the project folder name is used.
