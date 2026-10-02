# PLAN - fix-android-emulator-vscode
Task: Fix "Error fetching your Android emulators" from diemasmichiels.emulate extension on Linux

## Diagnosis (Phase 0)
- Extension: diemasmichiels.emulate-1.8.2 (`Android iOS Emulator`)
- Config key default: `emulator.emulatorPath = ~/Library/Android/sdk/emulator` (macOS path) — fails on Linux
- Correct Linux key: `emulator.emulatorPathLinux` should be `/home/rtx/Android/Sdk/emulator`
- SDK exists: /home/rtx/Android/Sdk/emulator/emulator (executable) ✅
- `emulator -list-avds` exits 0 with EMPTY output → zero AVDs (second root cause)
- `~/.android/avd/` missing → confirms no AVDs
- ANDROID_HOME / ANDROID_SDK_ROOT unset; `emulator` not on PATH
- cmdline-tools/ missing → no avdmanager/sdkmanager CLI; Studio not installed
- /dev/kvm present (666) → acceleration OK
- RULES.md: absent — proceeding with P1/P2 only
- Expo 57.0.26 (major 57), Expo Router project, no android/ dir (CNG)

## Subtasks
1. [orchestrator] Fix `.vscode/settings.json` → add `emulator.emulatorPathLinux` (tdd:false)
   - Verify: cat settings.json + `emulator -list-avds` no longer errors via wrong path
2. [orchestrator] Set ANDROID_HOME/SDK_ROOT + PATH in ~/.zshrc (tdd:false)
   - Verify: `echo $ANDROID_HOME` + `emulator -version`
3. [user-action OR auto] Create at least one AVD (Pixel_9_API_36) via cmdline-tools
   - Verify: `/home/rtx/Android/Sdk/emulator/emulator -list-avds` lists 1 AVD
4. [verification] Full gate: typecheck/lint not applicable (no code changed); evidence = emulator list + adb + extension path

## Edit order
.vscode/settings.json → ~/.zshrc → AVD creation (depends on cmdline-tools install)
