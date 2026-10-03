# Building on macOS 26/27 (Xcode 27, Apple Silicon)

Working notes from building v0.1.1 from a clean clone (MacBook Air M4, macOS 27,
Xcode 27, AppleClang 21). `docs/BUILDING.md` stays the reference; this branch only adds
what a current macOS toolchain needed on top of it. Not an official recipe.

## What this branch changes
- `build/fex-ios/build.sh`: pass `-DCMAKE_SYSTEM_PROCESSOR=arm64` (empty for iOS),
  `-DTUNE_ARCH=generic -DTUNE_CPU=none` (otherwise it reads /proc/cpuinfo) and
  `-DFEX_IOS_HOST` in the C/C++/ASM flags; also builds `JemallocLibs`.
- `build/fex-ios/fex-arm64-mac.patch`: `FEXCore/Source/Utils/ArchHelpers/Arm64.cpp` uses
  Windows-only types outside `#ifdef _WIN32`. FEX is a submodule, so this is a patch file:
  `git -C FEX apply ../build/fex-ios/fex-arm64-mac.patch`
- `build/wineserver/build.sh`: builds a base `libwineserver.a` from `wine/server` when none exists.
- `build/dxmt-ios/build.sh`: generates the airconv shader headers; Xcode 27's Metal compiler
  changed the atomic builtin, so the public `atomic_fetch_add_explicit` is used.
- `app/Madeira/JITAllocator.c`: inert stand-ins for FEX iOS-host symbols (`ios_fex_*`,
  `rpm_cas_snapshot_take`, `IosSubfloorToReal`, `IosMonoResolveRW`) referenced by FEXBridge.
- `SteamAuthAPI.swift`: retry transient URLSession errors (-1005 etc.) in the sign-in poll.

## Inputs that are not in git
- `toolchains/llvm-ios-build` (LLVM 15 static libs for iOS), `toolchains/llvm-mingw-*`,
  `toolchains/gnutls-ios`, `toolchains/ffmpeg-ios`
- `research/freetype`: `git clone --depth 1 --branch VER-2-13-3 https://github.com/freetype/freetype.git research/freetype`
- `brew install gnutls` (host headers for wine's config.h)
- `app/Madeira/x86_64-vcruntime`: 12 Microsoft DLLs, see `tools/fetch-vcruntime.md`.
  Not redistributable: do not commit them or publish an IPA that contains them.
- `app/Madeira/i386-windows`: built by `build/wine-i386/build.sh`.

## Order that worked
fex-ios -> wine config (`bash build/wine-pe/build-ntdll.sh`, `make -C wine/build-arm64ec/include`,
then `git checkout app/Madeira/arm64ec-windows/ntdll.dll` to restore the shipped file;
`ln -s build-arm64ec wine/build-macos`) -> wineserver -> ntdll-unix -> win32u-unix ->
dxmt-ios -> wine-i386 -> Xcode:

    xcodebuild -project app/Madeira.xcodeproj -scheme Madeira -configuration Debug \
      -destination 'generic/platform=iOS' -derivedDataPath ~/madeira-dd \
      CODE_SIGNING_ALLOWED=NO build

then zip `Payload/Madeira.app` into an IPA.

## Runtime notes (v0.1.1, observed in logs)
- Use a fresh LiveContainer data container: a prefix that lived under older builds crashed
  explorer when the Start button was pressed; a clean one did not.
- Put a `madeira.cfg` in the container's Documents (see `madeira.cfg.example`) with
  `env.MADEIRA_WOW_PLACEHOLDERS=1`. Without it the 32-bit SteamSetup.exe exits with 0xC0000005
  on a clean prefix.
- The JIT pool size varies per launch (560-624 MB here). At 560 MB the Steam client can
  exhaust it ("JIT pool exhausted") and crash; relaunching usually gives a larger pool.

## Additional requirements for v0.1.3

### Rust (on-device pairing library)

`app/Madeira/libmadeira_rppairing.a` is a Rust static library (idevice). Build it before `xcodebuild`:

~~~
brew install rustup
rustup default stable
rustup target add aarch64-apple-ios
export PATH="$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH"   # rustup's cargo first
bash build/rppairing-ios/build.sh
~~~

If Homebrew's `rust` is also installed, its `cargo` has no iOS standard library and the build fails with
`can't find crate for core`. Make sure `which cargo` points into `~/.rustup`.
The script also regenerates `app/Madeira/legal/LICENSES-rppairing-crates.txt`.

### LLVM headers for DXMT

`build/dxmt-ios/build.sh` (airconv, madeira_ags) needs `toolchains/llvm-project/llvm/include`
(LLVM 15 headers) next to the prebuilt `toolchains/llvm-ios-build`. Without it 18 files fail with
`'llvm/IR/Constants.h' file not found`.

### Rebuild order after updating

~~~
bash build/ntdll-unix/build.sh
bash build/wineserver/build.sh
bash build/dxmt-ios/build.sh
bash build/rppairing-ios/build.sh
xcodebuild ...   # then package the IPA
~~~

Run `build/fex-ios/build.sh` again only when the FEX submodule or `fex-arm64-mac.patch` changes.
Check for `BUILD SUCCEEDED` in the xcodebuild log before packaging: a failed link still leaves a partial `Madeira.app`.

### Entitlements (needed for Memory+)

The build uses `CODE_SIGNING_ALLOWED=NO`, so no entitlements are embedded. Embed them before zipping,
otherwise Memory+ (increased memory limit) stays off when installed through LiveContainer:

~~~
brew install ldid
ldid -S app/Madeira/Madeira.entitlements ~/ipa-work/Payload/Madeira.app/Madeira
~~~

### Choosing a JIT setup

- **LiveContainer + StikDebug**: JIT works; Memory+ comes from the LiveContainer host (e.g. the "Get More RAM" app). The built-in JIT of 0.1.3 is unavailable inside LiveContainer.
- **Direct install (SideStore/iLoader)**: built-in JIT works, but a free Apple ID does not grant the increased memory limit, so Memory+ stays off.
