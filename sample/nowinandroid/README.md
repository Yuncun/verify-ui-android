# Worked example — Now in Android

The config and Maestro flows used to verify two screens of Google's
[Now in Android](https://github.com/android/nowinandroid) (`demoDebug`) on a
Pixel_9 emulator. After `/omakase init`, these live at `.omakase/verify-ui.yml`
and `.omakase/verify-ui/*.yaml` in the adopter's repo.

- `verify-ui.yml` — two screens: For You, Interests
- `verify-ui/foryou.yaml` — launch, grant permissions, screenshot the For You screen
- `verify-ui/interests.yaml` — launch, tap Interests, screenshot the topic list
