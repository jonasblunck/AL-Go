# Test: Comprehensive Edge Cases

**Test ID:** edge-cases-20260401-130454  
**Branch:** `private/jonasbl/test-edge-cases-20260401-130454` → PR #6  
**Repo:** `jonasblunck/ALGoPTE-AgentTest`  
**Purpose:** Validate that positive scenarios pass and negative scenarios fail as expected across
`failOn`, `preprocessorSymbols`, and `buildModes` features.

---

## Test Matrix

| Project | Scenario | Key Setting | Expected Job | Actual Job |
|---------|----------|-------------|:---:|:---:|
| `POS-CleanWarning` | `failOn:warning` with zero-warning code | `failOn: "warning"` | ✅ PASS | TBD |
| `POS-MultiSymbols` | Two preprocessor symbols both active | `preprocessorSymbols: ["FEAT_A","FEAT_B"]` | ✅ PASS | TBD |
| `POS-SymbolElse` | Symbol not defined → `#else` branch compiles | `preprocessorSymbols: []` | ✅ PASS | TBD |
| `POS-CustomMode` | Custom build mode `Staging` | `buildModes: ["Staging"]` | ✅ PASS | TBD |
| `NEG-SyntaxError` | Missing `)` in `Message()` call | — | ❌ FAIL (AL0104) | TBD |
| `NEG-UndeclaredProc` | Calls non-existent procedure | — | ❌ FAIL (AL0132) | TBD |
| `NEG-FailOnWarning` | `[Obsolete]` pending call + `failOn:warning` | `failOn: "warning"` | ❌ FAIL (AL0432) | TBD |

**Overall PR Build expected: FAIL** (3 NEG projects fail by design)

---

## Project Details

### POS-CleanWarning
- App: codeunit 57000 "EC CleanWarning App"  
- Test: codeunit 57010 "EC CleanWarning Tests" — calls `Add(2,3)`, asserts = 5  
- Settings: `failOn: "warning"`, clean code with no obsolete calls  
- Validates: `failOn:warning` doesn't false-positive on warning-free code

### POS-MultiSymbols
- App: codeunit 57100 "EC MultiSymbol App"  
- Feature flags: `#if FEAT_A ... #endif` and `#if FEAT_B ... #endif`  
- Settings: `preprocessorSymbols: ["FEAT_A", "FEAT_B"]`  
- Validates: Multiple symbols can be active simultaneously

### POS-SymbolElse
- App: codeunit 57200 "EC SymbolElse App"  
- `#if EXPERIMENTAL_OFF` / `#else` → else branch compiles when symbol undefined  
- Settings: `preprocessorSymbols: []` (EXPERIMENTAL_OFF not defined → else branch active)  
- Validates: `#else` path compiles correctly when symbol is absent

### POS-CustomMode
- App: codeunit 57300 "EC CustomMode App"  
- Settings: `buildModes: ["Staging"]`  
- Validates: Custom build mode name works without errors; artifact named `*-Staging-Apps-*`

### NEG-SyntaxError
- App: codeunit 57400 with `Message('Hello'` ← missing `)`  
- Validates: AL0104 syntax error is properly detected and fails the build

### NEG-UndeclaredProc
- App: codeunit 57500 that calls `ThisProcedureDoesNotExist()`  
- Validates: AL0132 undeclared name error fails the build

### NEG-FailOnWarning
- App: codeunit 57600 calling an `[Obsolete('msg', 'pending')]` procedure  
- Settings: `failOn: "warning"`  
- Validates: AL0432 warning is treated as a failure (correct behavior)

---

## Runs

### Run 1 — Initial attempt
- **Run ID:** `23845527770`  
- **Result:** ❌ FAILED in Initialization  
- **Error:** `The build depth is too deep, the maximum build depth is 1. You need to run 'Update AL-Go System Files'`  
- **Root cause:** `main` branch has `TP` project (stage 2, depends on `BP`), but `PullRequestHandler.yaml` still had `workflowDepth: 1`. "Update AL-Go System Files" was failing in `@preview` due to a different bug (see [e2etest-test-projects.md](e2etest-test-projects.md)).
- **Fix:** Manually patched `workflowDepth: 1 → 2` in `PullRequestHandler.yaml` and `CICD.yaml` directly on `main` branch (commit `b3cd81f`).

### Run 2 — After workflowDepth fix
- **Run ID:** `23845631147`  
- **Triggered:** retrigger commit `ff29dab`
- **Result:** ❌ FAILED — but with one unexpected failure

**Actual job results:**

| Project | Expected | Actual | Match? |
|---------|----------|--------|:---:|
| `POS-CleanWarning` | ✅ PASS | ✅ PASS | ✅ |
| `POS-MultiSymbols` | ✅ PASS | ✅ PASS | ✅ |
| `POS-SymbolElse` | ✅ PASS | ✅ PASS | ✅ |
| `POS-CustomMode` (Staging) | ✅ PASS | ✅ PASS | ✅ |
| `NEG-FailOnWarning` | ❌ FAIL | ❌ FAIL (AL0432) | ✅ |
| `NEG-SyntaxError` | ❌ FAIL | ❌ FAIL (AL0104) | ✅ |
| `NEG-UndeclaredProc` | ❌ FAIL | ❌ FAIL (AL0132) | ✅ |
| `. (root)` | ✅ PASS | ❌ FAIL | ⚠️ **Unexpected** |

**POS-CustomMode artifact:** `POS-CustomMode-*-StagingApps-*` ✅ confirmed

---

## Unexpected Finding: Root Project Auto-Detection Bug

**The root `.` project failed because it compiled `NEG-SyntaxError` and `NEG-FailOnWarning` apps!**

From `BuildOutput.txt` of the `.` project:
```
Compilation started for project 'BP-Calculator' ...          → success
Compilation started for project 'BP-Calculator Tests' ...    → success
Compilation started for project 'NEG-FailOnWarning-App' ...  → AL0432 warning
Compilation started for project 'NEG-SyntaxError-App' ...    → AL0104 error ← FAIL
```

**Root cause:** `.AL-Go/settings.json` has `"appFolders": []` (empty array). In AL-Go, an empty
`appFolders` array triggers **auto-detection** — it scans ALL subdirectories for `app.json` files.
This finds `NEG-SyntaxError/App/app.json` and `NEG-FailOnWarning/App/app.json` even though those
subdirectories have their own `.AL-Go/settings.json` (making them separate projects). AL-Go does
**not** exclude sub-project folders from the parent project's auto-detection scan.

The same behavior also made the root project compile `BP/BP-App/` and `BP/BP-Tests/` — redundantly
alongside the dedicated `BP` project job.

**Impact:** The `failOn: "warning"` on `NEG-FailOnWarning` project caused the ROOT project to also
fail on the AL0432 warning it picked up, in addition to the AL0104 syntax error from NEG-SyntaxError.

**Fix:** To prevent the root project from auto-detecting apps, either:
- Set `"appFolders": ["__nonexistent__"]` in root `.AL-Go/settings.json` to explicitly provide no
  valid folders (prevents auto-scan), OR
- Remove root `.AL-Go/settings.json` entirely (root project ceases to exist)

This is a **test setup issue** (not an AL-Go bug), but it's a subtle and important behavior:
`appFolders: []` means "auto-detect", NOT "no app folders". Always set explicit app folders or
remove the root `.AL-Go/settings.json` in multi-project repos to avoid this.
