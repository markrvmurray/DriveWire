# Building DriveWire

Local notes for building the DriveWire host apps with a current Xcode (27) on
Apple Silicon, without an Apple Development certificate. The Swift app in
`swift/` is the more complete one; the Objective-C app in `objc/` is the older
one.

## Swift app

All commands run from `swift/`. There is no CocoaPods step: the Swift packages
(ORSSerialPort, swift-log, CocoaLumberjack, swift-argument-parser) are resolved
by Xcode on the first build.

### Build the GUI app and the CLI

```sh
xcodebuild -quiet -project DriveWire.xcodeproj -scheme DriveWire \
  -configuration Release \
  -derivedDataPath ~/Library/Developer/Xcode/DerivedData/DriveWire-swift \
  CODE_SIGN_IDENTITY=- \
  build

xcodebuild -quiet -project DriveWire.xcodeproj -scheme drivewire-cli \
  -configuration Release \
  -derivedDataPath ~/Library/Developer/Xcode/DerivedData/DriveWire-swift \
  CODE_SIGN_IDENTITY=- \
  build
```

Both products land in
`~/Library/Developer/Xcode/DerivedData/DriveWire-swift/Build/Products/Release/`:

```sh
open ~/Library/Developer/Xcode/DerivedData/DriveWire-swift/Build/Products/Release/DriveWire.app
~/Library/Developer/Xcode/DerivedData/DriveWire-swift/Build/Products/Release/drivewire-cli --help
```

Why each setting:

- **`-project`:** there is no workspace; the packages come in through the
  project itself.
- **`Release` is safe here:** unlike the Objective-C app, there is no appcast
  script, so a Release build doesn't touch any tracked files.
- **`-derivedDataPath` under `~/Library`:** same iCloud Drive problem as the
  Objective-C app. A build folder under `~/Documents` fails with "resource
  fork, Finder information, or similar detritus not allowed" at the
  `codesign` step.
- **`CODE_SIGN_IDENTITY=-`:** ad-hoc signing. `DEVELOPMENT_TEAM` is already
  empty and `ENABLE_HARDENED_RUNTIME` is already `NO` in the project, so no
  further overrides are needed.

### Install

Quit DriveWire first with ⌘Q: closing its window leaves the app running.

```sh
P=~/Library/Developer/Xcode/DerivedData/DriveWire-swift/Build/Products/Release
rm -rf ~/Applications/DriveWire.app
ditto $P/DriveWire.app ~/Applications/DriveWire.app
install -m 755 $P/drivewire-cli ~/.local/bin/drivewire-cli
```

- **Replaces the Objective-C app:** both apps build as `DriveWire.app`, so
  this overwrites an Objective-C build installed there. They have different
  bundle IDs (`com.boisypitre.DriveWire` for Swift, `com.tee-boy.DriveWire`
  for Objective-C), so each keeps its own settings. To keep both, copy the
  Swift app under another name, such as `DriveWire Swift.app`; renaming the
  bundle folder doesn't break the ad-hoc signature.
- **`rm -rf` first:** `ditto` merges into an existing bundle, which could
  leave stale files from an older build behind.
- **`ditto`, not `cp -r`:** it keeps the bundle's symlinks and metadata intact.
- **`~/.local/bin`:** the CLI's `INSTALL_PATH` is `/usr/local/bin`, which is
  owned by root; `~/.local/bin` is on `PATH` and needs no `sudo`.

### Notes

- **xcodebuild noise:** warnings about the `DVTCoreDeviceCore` plug-in,
  "CoreSimulator is out of date", and "command failed with exit code 0" come
  from the Xcode install and don't affect the macOS build. Check the exit
  status instead.
- **No drive sounds:** the Swift app has no insert/eject sounds. Its only audio
  is the built-in MIDI synth, used when the guest writes to `/midi`.

## Objective-C app

All commands run from `objc/`.

### 1. Install CocoaPods

```sh
brew install cocoapods
```

Homebrew's CocoaPods bundles its own Ruby; the system Ruby (2.6) is too old
for `sudo gem install`.

### 2. Podfile

The Podfile needs three changes from upstream:

- `source 'https://cdn.cocoapods.org/'` instead of the GitHub Specs repo, which
  would otherwise clone the whole multi-GB specs repository.
- `platform :osx, '12.0'`: the Xcode 27 SDK only accepts deployment targets
  from 12.0 to 27.0.x.
- A `post_install` hook forcing every pod target to 12.0 as well. Each pod
  otherwise keeps its own podspec minimum (CocoaAsyncSocket 10.8, Sparkle and
  CocoaLumberjack 10.13), which the SDK rejects:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['MACOSX_DEPLOYMENT_TARGET'] = '12.0'
    end
  end
end
```

`DriveWire.xcodeproj` also has `MACOSX_DEPLOYMENT_TARGET` raised to 12.0 in
all its build configurations.

### 3. Install the pods

```sh
pod install
```

This creates `Pods/`, `Podfile.lock` and `DriveWire.xcworkspace`. None of them
are committed upstream, so don't `git add` them.

### 4. Build and install into ~/Applications

```sh
xcodebuild -workspace DriveWire.xcworkspace -scheme DriveWire \
  -configuration Debug \
  -derivedDataPath ~/Library/Developer/Xcode/DerivedData/DriveWire-objc \
  DSTROOT=$HOME STRIP_INSTALLED_PRODUCT=NO \
  CODE_SIGN_IDENTITY=- DEVELOPMENT_TEAM= ENABLE_HARDENED_RUNTIME=NO \
  install

open ~/Applications/DriveWire.app
```

Quit DriveWire before rebuilding. Why each setting:

- **`-workspace`, not `-project`:** the app links `libPods-DriveWire.a`, which
  only the workspace builds.
- **`Debug`:** a Release build runs the "Set Appcast" script, which zips the app,
  runs Sparkle's `generate_appcast` (it needs the upstream signing key) and
  overwrites the tracked `DriveWireAppcast.xml`.
- **`-derivedDataPath` under `~/Library`:** `~/Documents` is synced by iCloud
  Drive, which puts `com.apple.FinderInfo` and file-provider attributes on the
  bundle folders as they are created, and `codesign` refuses to sign them.
  Stripping them with `xattr -cr` doesn't last, because iCloud puts them back.
- **`CODE_SIGN_IDENTITY=-` and `DEVELOPMENT_TEAM=`:** ad-hoc ("Sign to Run
  Locally") signing, because the keychain has no Apple Development identity.
- **`ENABLE_HARDENED_RUNTIME=NO`:** hardened runtime turns on library
  validation. Ad-hoc signatures have no Team ID, so dyld refuses the embedded
  Sparkle.framework ("different Team IDs") and the app dies at launch.
- **`install` with `DSTROOT=$HOME`:** `INSTALL_PATH` is `/Applications`, so the
  app lands in `~/Applications`. That stops LetsMove (`PFMoveApplication`)
  offering to move it on every launch. The Pods targets have
  `SKIP_INSTALL = YES`, so only the app is installed.
- **`STRIP_INSTALLED_PRODUCT=NO`:** `install` strips debug symbols by default.

### Notes

- **Sparkle updates:** the feed still points at the upstream GitHub releases.
  Decline automatic update checks, or Sparkle will replace this build with the
  upstream release.
- **Schemes and targets:** list them with
  `xcodebuild -list -workspace DriveWire.xcworkspace`.
- **Serial ports:** the app lists ports from IOKit, so a `usim09pt --aux-pty`
  pseudo-terminal (`/dev/ttysNNN`) probably won't appear in it.
