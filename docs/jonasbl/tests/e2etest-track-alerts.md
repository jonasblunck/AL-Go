# Test: trackALAlertsInGitHub — SARIF ErrorLogs Artifact

**Test ID:** track-alerts-20260401-120821  
**Branch:** `private/jonasbl/test-track-alerts-20260401-120821` → PR #10  
**Repo:** `jonasblunck/ALGoPTE-AgentTest`  
**Purpose:** Verify that `trackALAlertsInGitHub: true` causes AL compiler warnings to be captured
as SARIF output and uploaded as an `ErrorLogs` artifact. Build must still pass (`failOn: "error"`).

---

## Feature Under Test

`trackALAlertsInGitHub` in project-level `.AL-Go/settings.json`:
```json
{
  "trackALAlertsInGitHub": true,
  "failOn": "error"
}
```

**How it works:**
- Passes `-generateErrorLog:$true` to `Run-AlPipeline` (in `RunPipeline.ps1`)
- AL compiler generates a JSON error log alongside the build output
- `ProcessALCodeAnalysisLogs` action converts this to SARIF format
- SARIF is uploaded as `ErrorLogs` artifact
- With GitHub Advanced Security: SARIF gets uploaded to Code Scanning → alerts in Security tab + PR annotations
- Without Advanced Security: SARIF is only in the artifact (no security tab integration)

**Implementation reference:**
- `Actions/RunPipeline/RunPipeline.ps1` line 523: `-generateErrorLog:$settings.trackALAlertsInGitHub`
- `Actions/ProcessALCodeAnalysisLogs/ProcessALCodeAnalysisLogs.ps1`: SARIF generation logic
- `Actions/ProcessALCodeAnalysisLogs/baseSarif.json`: base SARIF template

---

## Project Structure

```
ALERTS/
  .AL-Go/settings.json    (trackALAlertsInGitHub: true, failOn: "error")
  ALERTS-App/
    AlertsApp.al          (codeunit 59300 with [Obsolete] calls → AL0432 warnings)
    app.json
```

**App code on test branch** (two calls to obsolete method):
```al
procedure Run(): Integer
begin
    // Two AL0432 warnings generated — both appear in SARIF output
    exit(OldCompute(5) + OldCompute(7));
end;
```

---

## Expected Behavior

| Aspect | Expected |
|--------|----------|
| Build result | ✅ **PASS** — `failOn: "error"` means AL0432 warnings don't fail |
| AL0432 in build log | ✅ Warnings appear in BuildOutput.txt |
| `ALERTS-*-ErrorLogs-*` artifact | ✅ **Generated** — contains SARIF JSON |
| SARIF content | AL0432 rule + 2 results with file/line/column for each call |
| GitHub Code Scanning | ❌ Not uploaded (repo likely lacks Advanced Security) |

---

## What to Verify

1. **Build passes**: ALERTS project job shows ✅ green
2. **ErrorLogs artifact exists**: Look for `ALERTS-*-ErrorLogs-*` in run artifacts
3. **Download and inspect SARIF**:
   ```bash
   gh run download <RUN_ID> --repo jonasblunck/ALGoPTE-AgentTest \
     --name "ALERTS-*-ErrorLogs-*" --dir /tmp/sarif-test
   cat /tmp/sarif-test/*.json | python3 -m json.tool | grep -A5 "ruleId"
   ```
4. Expected SARIF structure:
   ```json
   {
     "runs": [{
       "results": [
         { "ruleId": "AL0432", "locations": [{"physicalLocation": {"artifactLocation": {"uri": "AlertsApp.al"}, "region": {"startLine": ...}}}] },
         { "ruleId": "AL0432", "locations": [...] }
       ]
     }]
   }
   ```

---

## Runs

| Workflow | Run ID | Expected | Actual |
|----------|--------|----------|--------|
| PR Build (PR #10) | `23847841189` | PASS + ErrorLogs artifact | in_progress |
