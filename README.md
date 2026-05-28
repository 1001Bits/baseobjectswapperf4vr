# Base Object Swapper F4VR

VR port of [powerof3/BaseObjectSwapperF4](https://github.com/powerof3/BaseObjectSwapperF4) for Fallout 4 VR 1.2.72.

Runtime base-form / reference swaps and transform / property overrides via simple `_SWAP.ini` files. See the README inside the release archive for the INI syntax.

## Runtime requirements

- Fallout 4 VR 1.2.72
- [F4SEVR](https://f4se.silverlock.org/)
- [VR Address Library for F4SEVR](https://www.nexusmods.com/fallout4/mods/64879) — supplies `version-1-2-72-0.csv`
- Microsoft Visual C++ Redistributables 2019 (or any 2015–2022 combined)

> **Note**: this is the VR port. The NG version's BakaFramework requirement does **not** apply — the BOS code itself doesn't call into Baka, and no VR build of Baka exists.

## Build

```cmd
set VCPKG_ROOT=C:\vcpkg
set CommonLibF4Path=C:\path\to\CommonLibF4\with\VR\support
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

Output: `build/Release/BaseObjectSwapperVR.dll`.

CommonLibF4 must support `ENABLE_FALLOUT_VR` (the Buffout4-master fork works).

## Notes on the port

- VR-style `F4SEPlugin_Query` + `F4SEPlugin_Version` with `RUNTIME_LATEST_VR`
- `REL::Module::IsVR()` guard at load
- `TESObjectREFR::GetEncounterZone` (NG-only) replaced with `GetSaveParentCell()->GetEncounterZone()`
- Bounds-checked `formPair[1]` in `ObjectData::GetProperties` and `SwapFormData::GetForms` — malformed INI lines log an error and skip instead of crashing
- vcpkg pins `fmt 9.1.0` / `spdlog 1.11.0` (newer fmt breaks CommonLibF4's `REL::Version` formatter)

## Credits

powerofthree — original Base Object Swapper / Base Object Swapper F4.
