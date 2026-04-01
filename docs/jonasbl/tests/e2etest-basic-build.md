# Test: Basic Build and Test App

## What is being tested

Simplest possible AL PTE scenario: one main app (page extension) and one test app (codeunit
with `Subtype = Test`). Verifies that AL-Go can:
- Discover `appFolders` and `testFolders` from `.AL-Go/settings.json`
- Compile the main app and the test app
- Publish apps into a BC container and run AL unit tests
- Report test results in the workflow summary

## What to look for

- Pull Request Build conclusion: **success**
- Artifacts produced:
  - `ALGoPTE-AgentTest-main-Apps-*.zip` — compiled main app
  - `ALGoPTE-AgentTest-main-TestApps-*.zip` — compiled test app
- Jobs list includes a `Run tests` or `Build ALGoPTE-AgentTest` job with conclusion `success`
- Workflow summary shows at least **1 AL test passed**

## Test branch

`private/jonasbl/test-basic-build-20260401-104659` in `jonasblunck/ALGoPTE-AgentTest`

## Pull Request

https://github.com/jonasblunck/ALGoPTE-AgentTest/pull/1

## Triggered workflows

| Workflow | Run ID | URL | Status at trigger time |
|---|---|---|---|
| Pull Request Build | 23840197155 | https://github.com/jonasblunck/ALGoPTE-AgentTest/actions/runs/23840197155 | in_progress |

## App structure pushed

```
MyApp/
  app.json          (id: a1b2c3d4-e5f6-7890-abcd-ef1234567890, v1.0.0.0, idRange 50000-51000)
  HelloWorld.al     (pageextension 50000 on Customer List, shows message)
MyApp.Test/
  app.json          (id: b2c3d4e5-f6a7-8901-bcde-f12345678901, depends on MyApp)
  HelloWorld.Test.al (codeunit 60000 Subtype=Test, opens Customer List, asserts message shown)
.AL-Go/settings.json:
  country: "w1"
  appFolders: ["MyApp"]
  testFolders: ["MyApp.Test"]
```

## Notes

- First real test run through the agent instructions.
- Verifying the full instructions flow: clone → branch → app files → PR → run ID capture → doc.
- No `doNotPublishApps` or `useCompilerFolder` set, so full BC container pipeline runs.
- Expected to be slow (15–30 min) due to container spin-up on GitHub-hosted runner.
