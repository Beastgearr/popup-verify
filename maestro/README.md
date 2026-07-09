# Maestro flows — Claude-drivable iOS build verification (NEED-2)

These flows drive the EAS iOS **simulator** build through the demo walkthrough (login with an env-injected demo phone + OTP, then a tab-bar sweep) inside the `.github/workflows/sim-verify.yml` macOS Actions lane, which boots a simulator, installs the app, records video/screenshots, and uploads them as artifacts.

Demo credentials are never hardcoded in the flow files — `01-boot-login.yaml` references `${DEMO_PHONE}` / `${DEMO_OTP}`, injected at `maestro test` invocation time via `-e` flags (see the workflow's "Run Maestro walkthrough" step). This keeps the flows safe to sync verbatim into the public `popup-verify` mirror repo, which injects the same two values from GitHub Actions secrets instead of plaintext.

Trigger from Windows via gh CLI (omit `-f artifact_url` to use the build-#5 default; pass a fresh `applicationArchiveUrl` from `eas build:list --platform ios --json`):

```
gh workflow run sim-verify.yml -f artifact_url=https://expo.dev/artifacts/eas/<hash>.tar.gz
gh run list --workflow=sim-verify.yml   # then: gh run view <id> --log ; gh run download <id>
```

Artifacts land under the run's `sim-verify-<run_id>` bundle: `run.mp4` (screen recording), `final.png`, per-flow screenshots `01-*.png`…`08-*.png`, `maestro-debug/` (per-step logs + on-failure screenshots), and `maestro-report.xml` (JUnit).
