# Test: Preprocessor Symbols — MYFEATURE Conditional Compilation

## What is being tested

The `preprocessorSymbols` setting (v6.3), which specifies a list of preprocessor symbols
that will be active during AL compilation. When a symbol is declared, `#if SYMBOL` blocks
in AL code are compiled; the `#else` blocks are skipped.

This differs from `buildModes` + `conditionalSettings`: here, a single build mode
(Default) is used, but the symbols are set at the project level for that mode.

Key settings under test:
- `preprocessorSymbols: ["MYFEATURE"]` — activates the MYFEATURE symbol for all compilations
- `doNotPublishApps: true` — compile-only (fast)

The app uses `#if MYFEATURE` / `#else` in both a trigger and a layout section.
If the symbol is active, the extra field appears and the MYFEATURE message branch is compiled.

## What to look for

- PR Build conclusion: **success**
- Single build job: `Build . (Default)`
- The `MYFEATURE` code path is compiled — no compilation error from the conditional sections
- `*-Apps-*` artifact produced with one `.app` file
- No warnings related to the `#if` / `#else` blocks

## Test branch

`private/jonasbl/test-preprocessor-symbols-20260401-115051` in `jonasblunck/ALGoPTE-AgentTest`

## Pull Request

https://github.com/jonasblunck/ALGoPTE-AgentTest/pull/5

## Triggered workflows

| Workflow | Run ID | URL | Status |
|---|---|---|---|
| Pull Request Build | 23842750973 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23842750973 | in_progress |

## App structure pushed

`PS-App/` — pageextension 72000 "PS Customer List Ext"

```al
trigger OnOpenPage()
begin
#if MYFEATURE
    Message('MYFEATURE is ENABLED - conditional compilation working!');
#else
    Message('MYFEATURE is disabled - default path');
#endif
end;

#if MYFEATURE
layout
{
    addlast(Control1)
    {
        field(MyFeatureField; Rec.Name) { ... }
    }
}
#endif
```

`.AL-Go/settings.json`:
```json
{
  "preprocessorSymbols": ["MYFEATURE"],
  "doNotPublishApps": true
}
```

## Notes

- `preprocessorSymbols` can also be set in workflow-specific settings files
  (e.g., `.github/workflows/PullRequestHandler.settings.json`) to activate symbols
  only for certain workflows.
- Can also be set via `conditionalSettings` scoped to specific build modes:
  `{ "buildModes": ["Clean"], "settings": { "preprocessorSymbols": ["CLEAN"] } }`
- Multiple symbols can be listed: `["SYMBOL1", "SYMBOL2"]`
- If a symbol is NOT in `preprocessorSymbols`, `#if SYMBOL` blocks are skipped at compile time
