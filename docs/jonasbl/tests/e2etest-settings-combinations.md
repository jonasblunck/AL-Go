# Test: Settings Combinations — 4 Projects Mixing Multiple Settings

**Test ID:** settings-combinations-20260401-121410  
**Branch:** `private/jonasbl/test-settings-combinations-20260401-121410` → PR #11  
**Run ID:** `23848077183`  
**Repo:** `jonasblunck/ALGoPTE-AgentTest`  
**Purpose:** Verify that combinations of AL-Go settings work correctly together — no unexpected
interactions, no settings silently clobbering each other.

---

## Motivation

Individual settings have been tested in isolation. This test checks that settings don't interfere
when combined:
- Does `workspaceCompilation` still work when `buildModes` adds a second build pass?
- Does `conditionalSettings` correctly scope to a specific `buildMode` within a project?
- Do `preprocessorSymbols` in `testFolders` correctly activate the same symbols defined at project level?
- Does a project-level `failOn` override the repo-level `failOn`?
- Does `trackALAlertsInGitHub` generate SARIF for each build mode separately?

---

## Project Matrix

### COMBO-WS — `workspaceCompilation` + `buildModes` + `conditionalSettings` by buildMode

**Settings (`COMBO-WS/.AL-Go/settings.json`):**
```json
{
  "artifact": "////nextmajor",
  "workspaceCompilation": { "enabled": true, "parallelism": -1 },
  "buildModes": ["Default", "Clean"],
  "conditionalSettings": [
    { "buildModes": ["Clean"], "settings": { "preprocessorSymbols": ["CLEAN_MODE"] } }
  ]
}
```

**App structure:** `WS-Lib ← WS-App` (2-app dependency chain)

**What's combined:**
- Workspace compilation handles dependency resolution automatically
- Two build modes run in separate jobs (4 total: 2 apps × 2 modes — actually 2 jobs: 1 per mode with workspace handling both apps)
- `CLEAN_MODE` preprocessor symbol only active in Clean mode
- `WS-Lib.al` uses `#if CLEAN_MODE` to return different text per mode

**Expected jobs:**
| Job | Expected |
|-----|----------|
| `COMBO-WS (Default)` | ✅ PASS — `WS-Lib` returns `[Default]`, workspace compiles both apps |
| `COMBO-WS (Clean)` | ✅ PASS — `CLEAN_MODE` active, `WS-Lib` returns `[CLEAN]` |

---

### COMBO-BT — `appFolders` + `testFolders` + `preprocessorSymbols`

**Settings (`COMBO-BT/.AL-Go/settings.json`):**
```json
{
  "appFolders": ["BT-App"],
  "testFolders": ["BT-Tests"],
  "preprocessorSymbols": ["FEATURE_X", "FEATURE_Y"],
  "failOn": "error"
}
```

**What's combined:**
- Test app uses same `preprocessorSymbols` as app code (symbols propagate to `testFolders`)
- Tests assert on symbol-activated behavior: `GetFeatureStatus() == "active:XY"`, `GetMultiplier(5) == 50`
- Tests would FAIL if symbols weren't passed correctly to the test compiler

**Expected:**
| Aspect | Expected |
|--------|----------|
| App compilation | ✅ `#if FEATURE_X` and `#if FEATURE_Y` blocks compiled in |
| Test compilation | ✅ BT-Tests compiles against BT-App with symbols |
| Test run: `TestFeatureStatus` | ✅ `"active:XY"` — both symbols active |
| Test run: `TestMultiplierWithFeatureX` | ✅ returns `50` (5 × 10, FEATURE_X active) |

---

### COMBO-ALERTS — `buildModes` + `conditionalSettings` + `trackALAlertsInGitHub`

**Settings (`COMBO-ALERTS/.AL-Go/settings.json`):**
```json
{
  "buildModes": ["Default", "Strict"],
  "trackALAlertsInGitHub": true,
  "failOn": "error",
  "conditionalSettings": [
    { "buildModes": ["Strict"], "settings": { "failOn": "warning" } }
  ]
}
```

**What's combined:**
- `trackALAlertsInGitHub` generates SARIF — does it work per build mode?
- `conditionalSettings` scoped to `buildModes: ["Strict"]` overrides `failOn`
- Default mode: `failOn:error` (warning → PASS) + SARIF generated
- Strict mode: `failOn:warning` (same warning → FAIL) — quality gate pattern

**Expected:**
| Job | `failOn` applied | AL0432 | Expected |
|-----|-----------------|--------|----------|
| `COMBO-ALERTS (Default)` | `error` | Warning only | ✅ PASS + `ALERTS-*-ErrorLogs-*` artifact |
| `COMBO-ALERTS (Strict)` | `warning` (conditional) | Treated as error | ❌ FAIL (deliberate) |

---

### COMBO-OVERRIDE — project-level `failOn` overrides repo-level

**Settings (`COMBO-OVERRIDE/.AL-Go/settings.json`):**
```json
{
  "failOn": "none"
}
```

**Repo-level `.github/AL-Go-Settings.json`:** `failOn: "error"` (default)

**What's combined:**
- Project settings merge with (and override) repo settings
- `failOn: "none"` at project level should take precedence over repo `failOn: "error"`
- App has two `[Obsolete]` calls → two AL0432 warnings

**Expected:** ✅ PASS — `failOn:none` silences all diagnostics; warnings don't cause failure

---

## Overall Expected PR Result

| Project | Mode | Expected | Actual |
|---------|------|----------|--------|
| COMBO-WS | Default | ✅ PASS | TBD |
| COMBO-WS | Clean | ✅ PASS | TBD |
| COMBO-BT | Default | ✅ PASS (2 tests pass) | TBD |
| COMBO-ALERTS | Default | ✅ PASS + ErrorLogs | TBD |
| COMBO-ALERTS | Strict | ❌ FAIL (deliberate) | TBD |
| COMBO-OVERRIDE | Default | ✅ PASS | TBD |

**Overall PR Build: FAIL** (COMBO-ALERTS/Strict by design)

---

## Settings Interaction Map

```
workspaceCompilation ──────────────────────── artifact:nextmajor (required)
workspaceCompilation ──────────────────────── buildModes (each mode compiled via workspace)
buildModes ─────────────────────────────────── conditionalSettings (scoped by buildMode)
conditionalSettings ─────────────────────────── preprocessorSymbols (per mode)
conditionalSettings ─────────────────────────── failOn (quality gate per mode)
trackALAlertsInGitHub ──────────────────────── buildModes (SARIF per mode job)
preprocessorSymbols ─────────────────────────── appFolders + testFolders (symbols apply to both)
project failOn ──────────────────────────────── repo failOn (project overrides repo)
```
