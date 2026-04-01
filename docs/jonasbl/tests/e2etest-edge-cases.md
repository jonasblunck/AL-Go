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
- **Status:** In progress (triggered after retrigger commit `ff29dab`)  
- **Result:** TBD — update this section once complete

---

## What to Verify

1. Jobs `POS-*` all show ✅ in the GitHub Actions matrix
2. Jobs `NEG-*` all show ❌ in the matrix (expected failures)
3. Overall PR check shows as failed (any NEG failure causes PR check to fail)
4. For `POS-CustomMode`: verify artifact named `*-Staging-Apps-*` is uploaded
5. For `NEG-FailOnWarning`: verify error message mentions AL0432 in job logs
6. For `NEG-SyntaxError`: verify AL0104 in job logs
7. For `NEG-UndeclaredProc`: verify AL0132 in job logs
