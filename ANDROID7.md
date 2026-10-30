# The launcher on Android 7.0 / 7.1 (API 24/25)

This fork of [cleanyourtv](https://github.com/Gathin23/cleanyourtv) builds the reference launcher with
`minSdk 24`, so it installs on Android TV 7 sets that the upstream build refuses with
`INSTALL_FAILED_OLDER_SDK`. Everything else (the debloat lists, `tools/`, themes) is upstream's. The
debloat half needs none of this: it works on Android 7 from the stock home screen.

Tested on a Sony BRAVIA KD-55X8005C (2015, "BRAVIA 4K 2015", SVP-DTV15), Android TV 7.0, firmware
`SVP4KDTV15_EU-user 7.0 NRD91N.S44 5.457`: Home, the dock, the app grid, a theme with a wallpaper, Live TV
from the Inputs menu, and `power_on_home_always` with HomeKeyService, across a reboot. Other Android 7 sets
are untested; tell us in the issues if yours works or not.

## What changed

| File | Change |
|---|---|
| `launcher/app/build.gradle` | `minSdk 24`, and core library desugaring (`desugar_jdk_libs`) so `java.time` (API 26) is bundled into the APK |
| `MainActivity.java` | `PackageManager.getChangedPackages` (API 26) is skipped below 26: the tiles are rebuilt every time Home comes back, as the launcher did before that check existed |
| `UiFont.java` | `ResourcesCompat.getFont` instead of `Resources.getFont` (API 26) |
| `WallpaperCaptions.java` | `captions.txt` is read with a `BufferedReader` instead of `java.nio.file.Files` (API 26) |
| `WallpaperPlayer.java` | No `Bitmap.Config.HARDWARE` below 26; the software bitmap is kept |

`./gradlew lintRelease` reports zero errors at `minSdk 24`. The desugaring library makes the release APK
about 714 KB, over upstream's 576 KB gate in `.github/workflows/ci.yml`, so that CI check fails in this fork
by design; the TV it targets has 1.5 GB of RAM. On Android 7 the bundled Outfit font is not
used by the layouts (XML fonts need API 26), so text is drawn in the system font.

## Installing on Android 7

Debloat first, from the matching `examples/` list, exactly as upstream's README says. Then:

    launcher/build-and-install.sh            # or launcher\build-and-install.ps1 -TvHost <tv ip>

Two things behave differently on Android 7, and the script stops safely because of them:

1. `am start -W` answers `Status: timeout` even when the launcher opened fine, so the script reports
   "the new launcher did not start" and leaves the stock launcher enabled. Check for yourself:

       tools/tv.py 'dumpsys activity activities' | grep mFocusedActivity

   It should name `com.gathin.tvhome/.MainActivity`. A screenshot (`tools/tv.py --screenshot shots/home.png`)
   shows it drawn. If it is not there, stop: leave the stock launcher alone and open an issue.
2. `cmd package set-home-activity` prints nothing and changes nothing, so the stock launcher keeps the Home
   key. Once step 1 showed the new launcher running, hand Home over by disabling the stock launcher with a
   one-line list, so `tools/disable.sh` records it and `tools/restore.sh` can undo it. On the 2015 Sonys
   that is `com.google.android.leanbacklauncher`:

       printf 'com.google.android.leanbacklauncher\n' > work/stock-launcher.txt
       TV_HOST=<tv ip> tools/disable.sh work/stock-launcher.txt

   Press Home, check `mFocusedActivity` again, then reboot once and check that the TV comes back to the new
   Home before closing ADB.

The script's `--home-key-service` / `-HomeKeyService` step comes after the point where it stops, so enable a
key helper with the same shell line the script would send (see `Enable-Service` in
`launcher/build-and-install.ps1`), and check it is bound with `tools/tv.py 'dumpsys accessibility'`.

Back to the stock home:

    TV_HOST=<tv ip> tools/restore.sh work/stock-launcher.txt com.google.android.leanbacklauncher
