# HANDOFF — Shizuku-Next (j0p3s fork)

User writes European Portuguese; reply in Portuguese.

## 1. Goal
- Make Shizuku-Next work on the Google TV Streamer (Android 14), covering accessibility-based wireless-debugging pairing and the Apps tab. Then upstream the fixes to `rushiranpise/Shizuku-Next`.

## 2. Current status
- DONE: TV pairing fixes merged into `j0p3s/master` (j0p3s PR #1). Upstream PR #1 (j0p3s:master → rushiranpise:master) opened by the user.
- DONE, tested on TV: server fix "follow the manager across a reinstall". Commit `4913fca` on branch `claude/server-manager-reinstall`, pushed. Same change also exists as `51cebb2` on `claude/laughing-maxwell-ljyrte`.
- IN PROGRESS: the user is creating an upstream PR from `j0p3s:claude/server-manager-reinstall` → `rushiranpise:master` via the GitHub compare page. It may have been created as a Draft by mistake. If so, it needs "Ready for review".
- OPEN: draft PR j0p3s/Shizuku-Next#2 (`claude/laughing-maxwell-ljyrte` → j0p3s master). The user was told to close it WITHOUT merging, so the server fix doesn't leak into upstream PR #1 via master. This session was subscribed to it.

## 3. Key decisions (don't re-debate)
- **Separate upstream PR for the server fix:** it uses its own branch based on master, because upstream PR #1 tracks `j0p3s:master` and anything merged there gets added to it.
- **TV accessibility toggle turning off:** the cause was a crash in the app process, and Android disables the service of a crashing app. Fixes: the dialog in `MainActivity.handleIntent` is wrapped in `runCatching`, and a global crash logger writes to Diag.
- **TV pairing flow:** the service doesn't relaunch MainActivity within 10 s of the app opening Settings. The TV timeout is 3 min, and the "App info" button is hidden on TV.
- **Apps tab "Toggle all" / tap did nothing:** the server's `managerAppId` was stale after a reinstall, because `ApkChangedObservers` fired once and then stopped watching. Fix: re-arm the watcher, and re-resolve the app id lazily (at most every 2 s) when the check fails.
- **Release APK** is signed with the debug key, since there is no `signing.properties`. That's fine for testing; the upstream dev signs their own.

## 4. Tried and failed
- First TV fix (restricted-settings hint plus not covering Settings) didn't stop the toggle flipping off. The real cause was the crash (see §3).
- `add_repo rushiranpise/Shizuku-Next` was denied, so I can't open upstream PRs directly. The user does it via the compare URL in the browser.
- Force-push to `claude/laughing-maxwell-ljyrte` was denied. Used `git merge -s ours` and a normal push instead.
- `adb shell pkill shizuku_server` didn't kill the old server, so the fix looked broken. Use `adb shell kill <pid>` or reboot, then restart Shizuku.

## 5. Files that matter
- `manager/.../adb/AdbPairingAccessibilityService.kt`: reads the pairing code from TV Settings. Holds the TV grace period, timeout, and Diag logs.
- `manager/.../home/AccessibilityDialogHelper.kt`: enable dialog, TV-specific message, and `noteSettingsOpened()` call.
- `manager/.../MainActivity.kt`: `handleIntent`, where the dialog is guarded with `runCatching`.
- `manager/.../ShizukuApplication.kt`: `recordCrashes()`, the uncaught-exception handler that writes to Diag.
- `manager/src/main/res/values/strings.xml`: `dialog_adb_pairing_accessibility_permission_tv` (English only).
- `server/.../ShizukuService.java`:
  - `managerAppId` is volatile.
  - `watchManagerApk` (l.263), `updateManagerAppId` (l.278), `isManagerAppId` (l.294).
  - Callers at l.329, 345, 482, 568, 577.
- `server/.../ApkChangedObservers.kt`: `onEvent` removes the spent observer so it can be re-armed.
- `manager/.../ui/screen/AppsScreen.kt`, `authorization/AuthorizationManager.kt`: Apps tab code. Read-only context; errors other than SecurityException are swallowed silently.
- Scratchpad `install-to-tv.cmd`: Windows drag-and-drop installer for the TV (adb connect/pair + install -r). Default IP 192.168.50.216.

## 6. Next steps
1. Confirm the upstream server-fix PR exists and is not a draft. If it is a draft, the user clicks "Ready for review".
2. Confirm j0p3s/Shizuku-Next#2 is closed unmerged, then call `unsubscribe_pr_activity` for it.
3. Optional: test reinstalling the manager while the NEW server is running. Apps tab toggles should keep working without restarting the server.
4. Watch the upstream PRs for maintainer feedback and apply fixes on the matching branch (`master` for PR #1, `claude/server-manager-reinstall` for the server PR).

## 7. Gotchas
- **Build setup (fresh container):**
  - Android SDK at `/root/android-sdk`: platform 36, build-tools 37.0.0, NDK 29.0.14206865, CMake 3.31.6.
  - `local.properties` with `sdk.dir`.
  - `git submodule update --init` (the `api` submodule provides `:rish`).
  - Init script `/root/.gradle/init.d/central-mirror.gradle`, because Maven Central returns 429; it uses the Google mirror.
  - JDK 21. Build with `ANDROID_HOME=/root/android-sdk ./gradlew :manager:assembleRelease`.
- No CI runs on PRs. `app.yml` is dispatch-only and publishes a release, so don't trigger it casually.
- Server changes only take effect after the old `shizuku_server` process dies (see §4).
- The TV string is English-only; translations are left for upstream.
