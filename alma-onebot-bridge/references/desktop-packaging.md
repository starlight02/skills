# Desktop Packaging Reference

Build, sign, and ship the macOS menu bar app and the Windows tray app. These layers run the same Rust bridge binary in-process (Windows) or as a managed subprocess (macOS).

## Source Layout

```
platforms/
├── macos/                       SwiftUI menu bar app
│   ├── AlmaOneBotBridge/        Xcode project
│   ├── Assets.xcassets/         icons, accent color
│   ├── AppIcon.iconset/         source PNGs for icns generation
│   └── Localizable.xcstrings    UI strings (Simplified Chinese + English)
└── windows/                     Rust + WinUI tray app
    ├── src/app_state.rs         spawns the bridge on smol::block_on
    ├── src/i18n.rs              UI strings (Simplified Chinese + English),
    │                            selected by user locale with env fallback
    └── …
```

```
scripts/
├── build-macos.sh                       compile + xcodebuild + icon staging + ad-hoc sign
├── package-macos-pkg.sh                 produce a distributable PKG
├── package-windows-velopack.ps1         Velopack MSI
└── package-windows-zip.ps1              portable ZIP
```

## macOS Menu Bar App

The macOS app runs the Rust bridge as a child process so a crash in the binary does not kill the menu bar UI. It manages start/stop/restart, writes `~/.config/alma/bridge/config.toml`, opens the settings window, and tails the bridge log.

### Build

```bash
# Compile + xcodebuild + bundle the bridge binary into Resources/
./scripts/build-macos.sh

# Build and copy to /Applications so Launchpad lists it
INSTALL_TO_APPLICATIONS=1 ./scripts/build-macos.sh
```

Output: `platforms/macos/build/Build/Products/Release/AlmaOneBotBridge.app`.

### PKG Installer

```bash
./scripts/package-macos-pkg.sh
```

Local Apple Silicon-only validation:

```bash
BUILD_UNIVERSAL=0 TARGET=aarch64-apple-darwin PACKAGE_ARCH=arm64 \
  ./scripts/package-macos-pkg.sh
```

### PKG Invariants

These are non-negotiable; the GitHub Actions release workflow enforces them and broken PKGs silently install nothing.

- **Payload layout**: package from a staging root that contains `Applications/AlmaOneBotBridge.app`. The `distribution.xml` must use `install-location="/"`. Do **not** flip to `install-location="/Applications"` with a root-of-payload app — Installer.app silently skips installation in that combination.
- **Component plist must disable bundle relocation**:
  - `BundleIsVersionChecked = false`
  - `BundleIsRelocatable = false`
  - `BundleOverwriteAction = upgrade`
  - `RootRelativeBundlePath = Applications/AlmaOneBotBridge.app`
- **App icon**: source PNGs live in `platforms/macos/AppIcon.iconset/`. The build step generates `Contents/Resources/AppIcon.icns` with every standard macOS size including the 1024×1024 `icon_512x512@2x.png` rendition.
- **Icon wiring**: use `CFBundleIconFile=AppIcon.icns`. Delete `CFBundleIconName`. The asset-catalog icon path makes Launchpad wrap the icon in a system base, which makes it look smaller than native macOS icons.

### Versioning

Tags are immutable. If a release package changes after a tag is pushed:

1. Bump the patch in `Cargo.toml` (and the Xcode project, if it carries a separate marketing version).
2. Commit.
3. Create a new annotated tag, push it.

The scripts derive `CFBundleVersion` from semver as `major * 10000 + minor * 100 + patch`. Never move an existing tag.

### Verifying a Built PKG

After packaging:

```bash
pkgutil --expand-full dist/macos/AlmaOneBotBridge-*.pkg /tmp/pkg-check
ls /tmp/pkg-check/Payload/Applications/AlmaOneBotBridge.app/Contents/MacOS
/usr/libexec/PlistBuddy -c "Print :CFBundleIconFile" \
  /tmp/pkg-check/Payload/Applications/AlmaOneBotBridge.app/Contents/Info.plist
# expect: AppIcon.icns

# CFBundleIconName must not be set
/usr/libexec/PlistBuddy -c "Print :CFBundleIconName" \
  /tmp/pkg-check/Payload/Applications/AlmaOneBotBridge.app/Contents/Info.plist
# expect: Print: Entry, ":CFBundleIconName", Does Not Exist
```

## Windows Tray App

The Windows tray app is Rust + WinUI. Unlike the macOS app, the bridge runs **in-process** on `smol::block_on`, not as a child process. Keep `platforms/windows/src/app_state.rs` aligned with the runtime contracts in the root bridge — any Tokio-rooted dependency dragged in there would deadlock.

### Build

```powershell
# Velopack MSI installer
.\scripts\package-windows-velopack.ps1

# Portable ZIP
.\scripts\package-windows-zip.ps1
```

Cross-building from macOS or Linux validates the raw payload and ZIP. MSI packaging still requires a real Windows machine because Velopack runs Windows-side tooling.

### UI Localization

- Simplified Chinese and English variants are required for every visible string.
- macOS strings live in `platforms/macos/AlmaOneBotBridge/Localizable.xcstrings`.
- Windows strings live in `platforms/windows/src/i18n.rs`. Locale selection follows the Windows user locale with an environment-variable fallback (`ALMA_BRIDGE_LANG`).

## GitHub Actions Release Workflow

The workflow builds `arm64`, `amd64`, and `universal` artifacts on tag pushes. Two rules:

- Keep official actions on Node 24-compatible majors, currently `actions/checkout@v5` and `actions/upload-artifact@v5`. New deprecation warnings are addressed by upgrading the action, not by silencing the warning.
- Do **not** suppress Node runtime warnings with unsafe environment flags like `ACTIONS_ALLOW_USE_UNSECURE_NODE_VERSION`. They mask the actual deprecation and tend to break the next workflow run.
