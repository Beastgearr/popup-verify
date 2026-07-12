# Maestro flows — Claude-drivable iOS build verification (NEED-2)

These flows drive the EAS iOS **simulator** build through the demo walkthrough (login with an env-injected demo phone + OTP, a tab-bar sweep, a click-audit pass, and a no-location Moscow-fallback proof) inside the `.github/workflows/sim-verify.yml` macOS Actions lane, which boots a simulator, installs the app, records video/screenshots, and uploads them as artifacts.

Demo credentials are never hardcoded in the flow files — `01-boot-login.yaml` references `${DEMO_PHONE}` / `${DEMO_OTP}`, injected at `maestro test` invocation time via `-e` flags (see the workflow's "Run Maestro walkthrough" step). This keeps the flows safe to sync verbatim into the public `popup-verify` mirror repo, which injects the same two values from GitHub Actions secrets instead of plaintext.

## Flows

| File | What it drives | Depends on prior session? |
|---|---|---|
| `01-boot-login.yaml` | Cold-boot + demo login (Welcome → AuthMethods → PhoneEntry → OTP → MainTabs/Map). | No — `clearState: true`. |
| `02-map-and-tabs.yaml` | Post-login tab-bar sweep (Map → Friends → Settings → Map). | Yes — launches without `clearState`, relying on the persisted GoTrue session `01-boot-login.yaml` left behind. |
| `03-click-audit.yaml` | Click-audit smoke pass (v1-final-polish slice S5-b): filter chips, venue chip → `VenueDetailSheet` → create CTA → create sheet, FAB → create sheet, a friend row → `UserProfileScreen`, Settings' Help row → `FaqScreen` (+ back-out), a Legal row → `LegalDocumentScreen`. | No — self-contained via `runFlow: 01-boot-login.yaml`. Never taps `sheet-submit`, so it never creates a real pop-up against the live `popup-demo` backend; every opened sheet is dismissed with a pan-down-to-close swipe. |
| `04-no-location.yaml` | Moscow-fallback proof (v1-final-polish slice S5-c): login + map with **no simulated location at all** (not "permission denied" — the exact PopUp15 blank-void scenario), asserting `map-view` + stable chrome (`map-filter-chips` / `venue-carousel-band`) still render, proving S1-a's `MOSCOW_CENTER` camera fallback. | No — self-contained via `runFlow: 01-boot-login.yaml`. **Only reproduces the bug when the simulator's location has actually been cleared before this flow runs** — see "Location orchestration" below; Maestro itself cannot call `simctl`. |

## Location orchestration (`04-no-location.yaml`)

Maestro flows can't call `xcrun simctl`, so the no-location seed/clear has to happen at the CI-runner level, one layer up from the flow files:

- `flow: all` runs `01-boot-login.yaml` + `02-map-and-tabs.yaml` + `03-click-audit.yaml` as one `maestro test` invocation with the simulator's location seeded to Moscow (`55.7558,37.6173`, matching the app's own `MOSCOW_CENTER`), then runs `xcrun simctl location <UDID> clear` and invokes `04-no-location.yaml` as a **second, separate** `maestro test` invocation with no simulated location. Both invocations share the same job, video recording, and simulator boot; screenshots from each phase land under `artifacts/seeded/` and `artifacts/no-location/` respectively (both phases reuse `01-boot-login.yaml`'s screenshot names via `runFlow`, so they'd otherwise collide in the flat `artifacts/` dir).
- `flow: 04-no-location.yaml` (standalone dispatch) skips the Moscow seed entirely and clears any location up front, in the "Install app + seed location/permission" step.
- Any other single-flow dispatch (`01-boot-login.yaml`, `02-map-and-tabs.yaml`, `03-click-audit.yaml`) keeps the original single-invocation, Moscow-seeded behavior unchanged.

This logic is duplicated (by design, per the sync workflow) in `.github/workflows/sim-verify.yml` (private) and `scripts/mirror/sim-verify.yml` (public-mirror canonical source) — **both must be edited together**; `scripts/sync-verify-mirror.ps1` copies the latter into the public repo verbatim, so a private-only edit silently diverges the mirror.

## Char-drop hardening (`01-boot-login.yaml`)

Run `29037752215` showed Maestro's `inputText` drop 3 of 10 digits typing into the phone field on a loaded public runner (field ended up `9001111` → «Неверный формат»), plausibly worsened by the `NeonBubblesBackground` Reanimated organism running behind all 4 auth screens contending for the JS thread mid-type. After `inputText: ${DEMO_PHONE}`, the flow now asserts the phone field's rendered text (`phone-input-group-field-input`) actually holds the full value and, on mismatch, erases + retypes once via a conditional `runFlow: when: notVisible:` branch, then hard-asserts before continuing — deterministic, no fixed sleep added. A healthy run (green `29039997885`, 9m48s for this flow) never enters the retry branch.

Trigger from Windows via gh CLI (omit `-f artifact_url` to use the build-#5 default; pass a fresh `applicationArchiveUrl` from `eas build:list --platform ios --json`):

```
gh workflow run sim-verify.yml -f artifact_url=https://expo.dev/artifacts/eas/<hash>.tar.gz
gh run list --workflow=sim-verify.yml   # then: gh run view <id> --log ; gh run download <id>
```

**Free build lane (2026-07-12, EAS iOS quota exhausted):** the public mirror also carries `ios-build.yml` (canonical: `scripts/mirror/ios-build.yml`), which builds the unsigned Release simulator `.app` from the private source (read-only deploy key + Mapbox secrets on the mirror repo) and uploads it as the `popup-sim-build` artifact. Chain it into verification via the new `build_run_id` input (takes precedence over `artifact_url`):

```
gh workflow run ios-build.yml -R Beastgearr/popup-verify -f ref=main
gh run list -R Beastgearr/popup-verify --workflow=ios-build.yml        # grab <build_run_id> when green
gh workflow run sim-verify.yml -R Beastgearr/popup-verify -f build_run_id=<build_run_id> -f flow=all
gh run download <build_run_id> -n popup-sim-build -R Beastgearr/popup-verify   # local fetch for the bsdtar -> Sauce lane
```

Artifacts land under the run's `sim-verify-<run_id>` bundle: `run.mp4` (screen recording), `final.png`, per-flow screenshots `01-*.png`…`20-*.png` (under `seeded/` and `no-location/` subdirs when `flow=all`), `maestro-debug*/` (per-step logs + on-failure screenshots), and `maestro-report*.xml` (JUnit — split into `-seeded`/`-noloc` variants when `flow=all`).
