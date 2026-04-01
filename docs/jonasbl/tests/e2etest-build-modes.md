# Test: Build Modes — Default and Clean with Conditional Preprocessor Symbols

## What is being tested

The `buildModes` setting (introduced in AL-Go v2.3) combined with `conditionalSettings` to
assign preprocessor symbols per build mode (updated approach post-v6.3 deprecation of `cleanModePreprocessorSymbols`).

Key settings under test:
- `buildModes: ["Default", "Clean"]` — triggers two build matrix jobs
- `conditionalSettings` with `buildModes: ["Clean"]` → `preprocessorSymbols: ["CLEAN"]`
- AL app with `#if CLEAN` / `#else` conditional code blocks
- `doNotPublishApps: true` — compile-only (fast)

The PR Build runs a matrix with one job per build mode. Both modes must succeed.
Separate artifact sets are produced for each mode: `*-Apps-*` (Default) and `*-Clean-Apps-*`.

## What to look for

- PR Build conclusion: **success**
- Two build jobs in matrix:
  - `Build . (Default)` — compiles with no extra preprocessor symbols
  - `Build . (Clean)` — compiles with `CLEAN` preprocessor symbol active
- Artifacts: both `*-Apps-*` and `*-Clean-Apps-*` artifact sets uploaded
- Both `.app` files compile without errors
- `#if CLEAN` branch is active in Clean build, `#else` branch in Default

## Test branch

`private/jonasbl/test-build-modes-20260401-115051` in `jonasblunck/ALGoPTE-AgentTest`

## Pull Request

https://github.com/jonasblunck/ALGoPTE-AgentTest/pull/3

## Triggered workflows

| Workflow | Run ID | URL | Status |
|---|---|---|---|
| Pull Request Build | 23842736028 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23842736028 | in_progress |

## App structure pushed

`BM-App/` — pageextension 70000 "BM Customer List Ext"
- `#if CLEAN` → `Message('Running in CLEAN build mode')`
- `#else` → `Message('Running in Default build mode')`
- ID range: 70000–70099

`.AL-Go/settings.json`:
```json
{
  "buildModes": ["Default", "Clean"],
  "conditionalSettings": [
    {
      "buildModes": ["Clean"],
      "settings": {
        "preprocessorSymbols": ["CLEAN"]
      }
    }
  ],
  "doNotPublishApps": true
}
```

## Notes

- The deprecated `cleanModePreprocessorSymbols` setting was replaced in v6.3 by using
  `conditionalSettings` with `buildModes`. This test validates the new approach.
- Custom build modes (non-Default/Clean/Translated) also work with conditionalSettings.
- Artifact naming: Default mode → `<run>-Apps-<sha>`, Clean mode → `<run>-Clean-Apps-<sha>`
