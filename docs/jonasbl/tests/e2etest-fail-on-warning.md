# Test: failOn Warning — AL0432 Obsolete Call

## What is being tested

The `failOn` setting (v7.2) configured to `"warning"`, which causes the PR Build to fail
when the AL compiler emits any compiler warning (not just errors).

This verifies that:
1. AL-Go correctly detects compiler warnings (not just errors)
2. `failOn: "warning"` causes the build conclusion to be `failure`
3. The pipeline reports the correct reason (warning count, not compilation error)

Key settings under test:
- `failOn: "warning"` — fail the build if any compiler warning is emitted
- `doNotPublishApps: true` — compile-only (fast; container not needed for this)

The app deliberately contains an `[Obsolete('...', 'pending')]` procedure that is
called from a non-obsolete procedure. This triggers compiler warning **AL0432**:
> 'OldProc' is obsolete: 'Use NewProc instead - this will trigger AL0432 warning'

Other possible `failOn` values:
- `none` — never fail on warnings (only errors)
- `warning` — fail on any warning
- `newWarning` — fail on PR if new warnings were introduced (compares against CI/CD baseline)
- `error` — fail only on actual errors (default)

## What to look for

- PR Build conclusion: **failure** (this is the EXPECTED outcome)
- The build job `Build . (Default)` fails
- Build output logs mention: `AL0432` warning
- The failure is due to `failOn: warning` detecting the AL0432 warning
- The `Pull Request Status Check` job reports PR build failed
- **NOT** a compilation error — the code compiles successfully, but `failOn` triggers

## Test branch

`private/jonasbl/test-fail-on-warning-20260401-115051` in `jonasblunck/ALGoPTE-AgentTest`

## Pull Request

https://github.com/jonasblunck/ALGoPTE-AgentTest/pull/4

## Triggered workflows

| Workflow | Run ID | URL | Status |
|---|---|---|---|
| Pull Request Build | 23842741266 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23842741266 | in_progress |

## App structure pushed

`FOW-App/` — codeunit 71000 "FOW Warning Test"

```al
[Obsolete('Use NewProc instead - this will trigger AL0432 warning', 'pending')]
procedure OldProc()
begin
    // obsolete pending procedure
end;

procedure NewProc()
begin
    OldProc(); // ← triggers AL0432 warning
    Message('Test complete');
end;
```

`.AL-Go/settings.json`:
```json
{
  "failOn": "warning",
  "doNotPublishApps": true
}
```

## Notes

- `failOn: "newWarning"` (v7.2) is more sophisticated: it compares warnings from the
  PR build against the last successful CI/CD run on the target branch. New warnings fail
  the PR; existing warnings don't. Requires a CI/CD baseline run on main.
- This test uses `failOn: "warning"` (simpler — fail on any warning) to avoid needing
  a baseline CI/CD run.
- AL0432 is emitted for `ObsoleteState = Pending` (not just `Removed`), which is the
  mildest form of obsolescence and always produces a warning rather than error.
