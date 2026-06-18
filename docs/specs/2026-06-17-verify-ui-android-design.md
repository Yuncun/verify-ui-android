# verify-ui for Android — design

Status: approved 2026-06-17. The first specialized omakase harness source: an
adopter installs it into an Android repo and the agent cannot push UI work until
a screen's actual rendered pixels pass an AI judge.

## Problem

An agent claims an Android screen is "done." The only standing proof is that the
code compiled — not that the screen renders. Blank screens, crash dialogs, and
broken Material layouts survive "done" and reach the push. verify-ui makes the
rendered pixels a precondition for pushing.

## What is reused (the spine, unchanged)

omakase's deferred-gate machinery is UI-agnostic and reused verbatim:

- `deferred-check.sh` — push-time gate. Blocks the push unless a fresh PASS was
  recorded for the exact commit, scoped to changed UI paths. Does not run the
  judge; reads a record.
- `omakase-record.sh` — verdict recorder, keyed to HEAD, written under the git
  dir (never committed).
- `omakase-ledger.sh` / banner — the 🍣 scorecard.

verify-ui adds exactly one new thing: a producer.

## The producer — `/verify-ui`

A skill the agent runs at done-time (the agent is the judge). Procedure:

1. Read `.omakase/verify-ui.yml`.
2. Ensure a device: `adb devices`. None → record ERROR with guidance ("start an
   emulator"), print the scorecard, exit. Never cold-boot inside the gate.
3. Build + install the app — the config's `build` command.
4. For each configured screen, run its Maestro flow (launch → navigate →
   `takeScreenshot` to a known path).
5. Read each screenshot and judge PASS / FAIL / ERROR against the rubric below.
6. Print a one-line-per-screen scorecard.
7. Record the verdict (`omakase-record.sh --check verify-ui --verdict ...`).

Never-break discipline (inherited from visual-verify): any step failing marks
that screen ERROR with a one-line reason and the run continues — the scorecard
always prints, the verdict is always recorded, cleanup always runs.

## The judge rubric (per-screen render check, v1)

PASS iff the screenshot shows a real, intact screen:
- no system crash dialog ("app keeps stopping") or ANR,
- no blank/white/black void where content belongs,
- primary content present,
- no obviously broken Material layout (overlapping text, zero-size or unstyled
  components, raw placeholder text).

FAIL on crash / blank / broken. ERROR on harness failure (no device, build
failure, flow error — distinct from a screen that rendered wrong).

Explicitly NOT in v1: pixel-perfect Material-spec auditing (exact dp/tokens).
The check is "did it render a working screen," not "is it spec-compliant."

## Config — `.omakase/verify-ui.yml`

```yaml
device:
  avd: Pixel_9          # optional; omit to use whatever adb device is connected
build: "./gradlew :app:installDemoDebug"
package: com.example.app
screens:
  - name: Home
    flow: .omakase/verify-ui/home.yaml
  - name: Settings
    flow: .omakase/verify-ui/settings.yaml
```

Each `flow` is a Maestro YAML that launches the app, navigates to the screen, and
takes a screenshot to a path the producer reads.

## Gate wiring — `lefthook-local.yml` (pre-push)

```yaml
- name: deferred-check-verify-ui
  run: bash .omakase/bin/omakase-ledger.sh verify-ui -- bash .omakase/gates/deferred-check.sh
  env:
    OMAKASE_CHECK: verify-ui
    OMAKASE_GLOB: '*/src/*'   # the adopter's UI source paths
    OMAKASE_HOOK: pre-push
```

## Distribution

A standalone omakase source repo: `omakase.manifest` + a self-contained
`payload/` (base plumbing + deferred-gate scripts + the producer). Installed with
`/omakase init <git-url-or-path>`. Same proven path as pixterm-harness.

## Proof (this build)

Install the harness into a clone of Google's **Now in Android** (canonical
Compose / Material 3 reference app; `demoDebug` runs offline). On a Pixel_9
emulator: a real screen → PASS; a deliberately broken screen → FAIL → push
blocked; remove → clean working tree.

## Non-goals (v1)

- Stateful multi-step scenarios (a later "advanced mode").
- Native iOS; real-device farms.
- Cold-booting emulators inside the push gate.
- Pixel-perfect Material auditing.

## Follow-ups (recorded, not built)

- Promote `deferred-check.sh` + `omakase-record.sh` into the omakase BASE payload
  — they are generic and every source can use them.
- A web producer as a sibling source (the original plan), sharing the same spine.
- Finalize the published source name.
