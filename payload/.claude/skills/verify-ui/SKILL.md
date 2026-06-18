---
name: verify-ui
description: Best-effort visual verification of Android UI work. Builds + installs the app on a running emulator, drives each configured screen with a Maestro flow, screenshots it, judges PASS/FAIL/ERROR from the rendered pixels against a Material 3 render rubric, prints a one-line-per-screen scorecard, and records its verdict for the pre-push deferred gate. Invoke at done-time on UI work; the agent may self-invoke before claiming completion.
allowed-tools: Bash(adb *) Bash(maestro *) Bash(./gradlew *) Bash(git *) Bash(sed *) Bash(grep *) Bash(.omakase/bin/omakase-record.sh *) Bash(.omakase/bin/omakase-ledger.sh *) Read
context: fork
---

# /verify-ui — Android Visual Verification

You are the **Evaluator**. You did not write the code. Your job is to drive the
running app like a skeptical user and report what actually renders — not what the
code "should" do. If you catch yourself reasoning "the code looks fine," stop:
you are here to look at pixels, not read source.

You drive the UI and judge it; you print a scorecard and **record a verdict** that
the pre-push deferred gate reads (Step 6). You do not block anything yourself — the
gate enforces your recorded verdict at push time. The scorecard is for a human to
skim; the record is what the gate checks.

## The one rule: never break

The run must survive anything. The emulator missing, the build failing, one
Maestro flow erroring, a screen never appearing — none of that aborts the run.
Mark that screen **ERROR** with a one-line reason and move on. **The scorecard
always prints, the verdict is always recorded (Step 6), and cleanup (Step 7)
always runs.** A half-finished run that prints honest rows beats a clean crash
that prints nothing.

## Procedure

### 1. Orient

Read the config and see what changed so you know what you are verifying:

```bash
test -f .omakase/verify-ui.yml || { echo "no .omakase/verify-ui.yml — nothing configured to verify"; exit 0; }
cat .omakase/verify-ui.yml
git diff --stat "$(git merge-base origin/HEAD HEAD 2>/dev/null || echo HEAD~1)"..HEAD 2>/dev/null
```

If no UI-related code changed on this branch, say so and exit — there is nothing
to verify.

### 2. Ensure a device

```bash
export ANDROID_HOME="${ANDROID_HOME:-$HOME/Library/Android/sdk}"
export PATH="$ANDROID_HOME/platform-tools:$HOME/.maestro/bin:$PATH"
adb devices | grep -qw device || echo "NO DEVICE"
```

If no device is connected: if the config names an `avd`, you MAY boot it headless
(`"$ANDROID_HOME/emulator/emulator" -avd <name> -no-window -no-snapshot-save &`)
and wait for `adb wait-for-device` + `getprop sys.boot_completed == 1`. If you
cannot get a device, record ERROR with guidance ("start an emulator, then re-run
/verify-ui"), print the scorecard, and run cleanup. Never cold-boot inside a git
hook — only here, at done-time.

### 3. Build + install the app

Run the config's `build` command (default `./gradlew :app:installDemoDebug`).
Set `JAVA_HOME` if the project needs a specific JDK. If the build fails, record
ERROR for every screen with the build's last error line, then scorecard + cleanup.

### 4. Drive + screenshot each configured screen

For each `screens[]` entry, run its Maestro flow. Each flow launches the app,
handles first-run permission dialogs (`permissions: { all: allow }`), navigates to
the screen, and takes a screenshot. Run from a known directory so you know where
the PNG lands:

```bash
cd "$(git rev-parse --show-toplevel)"
maestro test <flow> 2>&1 | tail -20    # the flow's takeScreenshot writes <name>.png
```

If a flow errors (element never appears, app crashes mid-flow), that screen is
ERROR — but a crash that Maestro surfaces as "app not responding / stopped" is a
real render failure, judge it **FAIL**, not ERROR. When in doubt, look at whatever
screenshot was produced before the error.

### 5. Judge each screenshot

`Read` each PNG and judge against this rubric. PASS only if **all** hold:

- no system crash dialog ("<app> keeps stopping") or ANR,
- no blank/white/black void where content belongs,
- the screen's primary content is present (not just a bare app bar),
- no obviously broken Material layout — overlapping text, zero-size or unstyled
  components, raw placeholder/lorem text, a stack trace on screen.

Verdicts: **PASS** (rendered intact), **FAIL** (crash / blank / broken),
**ERROR** (the harness could not produce a usable screenshot — no device, build
failure, flow error). You are NOT auditing pixel-perfect Material spec compliance;
you are confirming the screen actually rendered a working UI.

### 6. Scorecard + record the verdict

Print one line per screen:

```
verify-ui — 2 screens
  PASS  For You (home)
  FAIL  Settings — blank content area below the app bar
```

The run verdict is **pass** only if every screen is PASS. Any FAIL → fail. An
ERROR-only run (nothing could be judged) is fail (the gate must not pass on a run
that proved nothing). Record it:

```bash
bash .omakase/bin/omakase-record.sh --check verify-ui --verdict pass   # or fail
```

To override a judged FAIL (you have a documented reason it is acceptable to push
anyway), record a waiver — it is surfaced loudly at push time, never silent:

```bash
bash .omakase/bin/omakase-record.sh --check verify-ui --verdict pass \
  --original-verdict fail --reason "Settings blank is a known upstream data stub, tracked in #123"
```

### 7. Cleanup

Force-stop the app so the next run starts clean. Leave the emulator running if it
was already up when you started; only shut down one you booted yourself.

```bash
adb shell am force-stop "$(sed -n 's/^package:[[:space:]]*//p' .omakase/verify-ui.yml | head -1)" 2>/dev/null || true
```
