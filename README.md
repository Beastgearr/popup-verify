# popup-verify

CI verification harness for [Pop Up](https://popupgram.app) demo builds. It boots an iOS simulator on a GitHub-hosted macOS runner, installs an EAS-built simulator `.app`, and drives it through Maestro flows -- recording video, screenshots, and crash reports as workflow artifacts.

This repo exists on the free GitHub public-repo macOS runner tier because the private application repo's own free Actions minutes are exhausted. **It contains no application source code** -- only the CI workflow and Maestro flow files, synced verbatim (with credentials stripped) from the private repo.

Runs are triggered manually via `workflow_dispatch` (`gh workflow run sim-verify.yml -f artifact_url=<EAS tar.gz URL> -f flow=all`), passing the URL of an EAS-built iOS simulator artifact. Demo login credentials are never stored in this repo -- they're injected at runtime from this repo's own `DEMO_PHONE` / `DEMO_OTP` Actions secrets.

The `maestro/` flows and `.github/workflows/sim-verify.yml` are kept in sync with the private repo via `scripts/sync-verify-mirror.ps1` (run from the private repo) -- do not hand-edit them here, edits will be overwritten on the next sync.
