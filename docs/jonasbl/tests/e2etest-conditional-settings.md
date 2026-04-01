# Test: conditionalSettings — Branch-Scoped failOn

**Test ID:** conditional-settings-20260401-120821  
**Repo:** `jonasblunck/ALGoPTE-AgentTest`  
**Purpose:** Verify that `conditionalSettings` with `branches` condition correctly scopes settings
to matching branches. Demonstrate the same code produces different build outcomes depending on branch.

---

## Feature Under Test

`conditionalSettings` in `.github/AL-Go-Settings.json`:
```json
{
  "conditionalSettings": [
    {
      "branches": ["feature/*"],
      "settings": {
        "failOn": "warning"
      }
    }
  ]
}
```

**How it works:**
- Supported condition keys: `branches`, `repositories`, `projects`, `workflows`, `users`
- All specified conditions in a block must match (AND logic)
- Multiple matching blocks are applied in order (later overrides earlier for scalars)
- Branch name is matched with glob-style patterns (`feature/*` matches `feature/foo`, `feature/bar`)

---

## Test Setup

**COND project** (added to main, `COND/.AL-Go/settings.json`):
```json
{ "country": "w1", "appFolders": ["COND-App"] }
```

**COND app on test branches** — adds an `[Obsolete]` call that generates **AL0432 warning**:
```al
procedure Run(): Integer
begin
    exit(GetValue());  // GetValue() is marked [Obsolete('pending')] → AL0432
end;
```

---

## Two Parallel PRs — Same Code, Different Branch

### PR #8: `feature/test-conditional-settings-20260401-120821` → **Expected: FAIL**

- Branch matches `feature/*` → conditionalSettings applies → `failOn: "warning"`
- AL0432 warning treated as a failure → build fails for COND project

### PR #9: `private/jonasbl/test-conditional-settings-20260401-120821` → **Expected: PASS**

- Branch matches `private/jonasbl/*` → no conditionalSettings match → default `failOn: "error"`
- AL0432 warning is just a warning → build passes despite the obsolete call

---

## What to Verify

1. PR #8 (feature/*): COND job **FAILS** with error message mentioning AL0432 / failOn:warning
2. PR #9 (private/*): COND job **PASSES** (warning in build output but not a failure)
3. In PR #8 job log: look for message like `Compiling... failOn=warning` or the AL0432 line causing failure
4. Both runs should show AL0432 warning in the BuildOutput artifact — only the branch/settings scope differs

---

## Runs

| PR | Branch | Run ID | Expected | Actual |
|----|--------|--------|----------|--------|
| PR #8 | `feature/test-conditional-settings-20260401-120821` | `23847840726` | ❌ FAIL (failOn:warning) | in_progress |
| PR #9 | `private/jonasbl/test-conditional-settings-20260401-120821` | `23847840857` | ✅ PASS (failOn:error) | in_progress |

---

## Additional conditionalSettings Options (Not Tested Here)

From source code analysis (`Tests/ReadSettings.Test.ps1`):

| Condition Key | Example | Matches on |
|---------------|---------|-----------|
| `branches` | `["feature/*", "release/*"]` | Current branch name |
| `repositories` | `["myapp-*"]` | Repository name |
| `projects` | `["build/*", "MyProject"]` | AL-Go project folder path |
| `workflows` | `["CI/CD", "PullRequestHandler"]` | Workflow name (sanitized) |
| `users` | `["alice", "bob"]` | GitHub actor username |

**AND logic:** Multiple keys in one block all must match. Multiple blocks are OR (any match applies).
