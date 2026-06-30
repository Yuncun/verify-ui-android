# verify-ui-android

An omakase harness source. An agent cannot push Android UI work until a screen's
rendered pixels pass an AI judge.

## Install

    /omakase init https://github.com/<you>/verify-ui-android

Edit `.omakase/verify-ui.yml` for your app, then write one Maestro flow per screen.

## Run

    /verify-ui

Builds and installs your app on a running emulator, screenshots each configured
screen, judges PASS/FAIL from the pixels, and records a pass when every screen
renders. `git push` is blocked until a fresh PASS exists for the commit; bypass a
single push (audited in the scorecard) with `OMAKASE_SKIP_VERIFY_UI=1`.

## Remove

    /omakase remove

## Requires

- Android SDK and a running emulator or device
- Maestro — https://get.maestro.mobile.dev

## Design

`docs/specs/2026-06-17-verify-ui-android-design.md`
