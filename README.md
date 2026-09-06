# GPS Survey — releases

Release builds of **GPS Survey**, an Android GNSS land-survey app for Israel.

This repository holds only the shipped artifacts and the manifest the app reads when you
tap **Check for update**. The source lives elsewhere.

## Install

Download the latest `.apk` from [Releases](../../releases/latest) and open it on the phone.
Android will ask you to allow installing from this source the first time.

Once installed, the app updates itself: **☰ → Settings → About → Check for update**.

## What the app reads

[`latest.json`](latest.json) — one small file, fetched over HTTPS:

| Field | Meaning |
|---|---|
| `versionCode` | Integer. The app compares this against its own; higher means an update exists. |
| `versionName` | What the user sees, e.g. `0.1.0`. |
| `apkUrl` | Direct download link for that version's APK. |
| `sizeBytes`, `sha256` | Checked after download; a mismatch aborts the install. |
| `mandatory` | If true the app nags rather than offering. Reserved for a build that fixes a data-corrupting bug. |
| `notes.en` / `notes.he` | Shown in the update dialog. |

## Updates only work if every build is signed with the same key

Android refuses to install an update signed with a different key than the installed app.
If a release goes out under a new key, every user has to **uninstall first — losing every
project on the device** — and reinstall.

So: one keystore, created once, backed up somewhere that survives a lost laptop. If it is
lost, no existing installation can ever be updated again.

## Cutting a release

From the source repository:

```bash
./gradlew :app:assembleRelease          # signed, if keystore.properties is present
gh release create v0.2.0 app/build/outputs/apk/release/app-release.apk \
  --repo maozwe/gps-survey-releases \
  --title "GPS Survey 0.2.0" --notes-file CHANGELOG-0.2.0.md
```

Then update `latest.json` here — `versionCode`, `versionName`, `apkUrl`, `sizeBytes`,
`sha256` (`sha256sum app-release.apk`) and the notes — and push. The app picks it up on the
next check.

`versionCode` must increase by exactly 1 per release, and must match the `versionCode` baked
into the APK you uploaded. The app trusts this file, so a wrong number here either hides a
real update or offers one that is already installed.

## Status

**0.1.0 has never been connected to a GNSS receiver.** Every parser and transform is tested
against reference data, and nothing has been tested against real hardware. Treat this build
as something to take to a known control point with a known receiver, not as something to
survey with.
