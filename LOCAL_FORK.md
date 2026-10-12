# Local fork — paste-last-transcript shortcut

This clone carries one feature that upstream will not merge, plus the build tweaks needed
to produce a Windows x64 installer outside cjpais's CI.

## The feature

A global shortcut that pastes the most recent transcript at the cursor — the tray's
_Copy last transcript_ action, but bound to a hotkey and pasting instead of copying.

- Binding id: `paste_last_transcript`
- Default: `alt+shift+z` (`option+shift+z` on macOS)
- Rebindable in **Settings → General**, directly under the Transcribe shortcut

| File                                                  | Change                                                                                                    |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/actions.rs`                            | `PasteLastTranscriptAction` + its `ACTION_MAP` entry                                                      |
| `src-tauri/src/settings.rs`                           | Default `ShortcutBinding` in `get_default_settings()`                                                     |
| `src-tauri/src/tray.rs`                               | `last_transcript_text` widened to `pub(crate)` so the tray and the shortcut share one text-selection rule |
| `src/components/settings/general/GeneralSettings.tsx` | `<ShortcutInput shortcutId="paste_last_transcript" />`                                                    |
| `src/i18n/locales/en/translation.json`                | Label and description; the other 23 locales fall back to English                                          |

Notes for future rebases:

- The shortcut plumbing (`shortcut/handler.rs`, `shortcut/mod.rs`, `shortcut/tauri_impl.rs`,
  `shortcut/handy_keys.rs`) is fully generic over `ShortcutBinding` and needs no changes.
- `utils::paste` is a re-export of `clipboard.rs::paste`. It drives Enigo, so it must be
  called inside `app.run_on_main_thread(...)` — skipping that fails silently.
- Existing installs pick the binding up automatically: `settings.rs` merges any default
  bindings missing from a stored settings file.

Upstream position: cjpais declined PR #659 and is collecting hotkey requests in
[discussion #211](https://github.com/cjpais/Handy/discussions/211) — "new keyboard shortcuts
are not going to be merged at this time". So this stays a fork.

## Building on Windows x64

Handy's Windows x64 target enables the `vulkan` whisper backend, so the build needs the
Vulkan SDK. It also needs SPIRV-Headers — but the SDK already ships
`SPIRV-HeadersConfig.cmake` in its `Lib/cmake`, so CI's separate vcpkg step is unnecessary.
The catch is that the config file sits directly in `Lib/cmake`, not in a `SPIRV-Headers`
subdirectory, so `CMAKE_PREFIX_PATH` must name `Lib` itself for `find_package` to resolve it.

One-time install:

```bash
winget install --id KhronosGroup.VulkanSDK --version 1.4.309.0
```

Every build:

```bash
export VULKAN_SDK="C:/VulkanSDK/1.4.309.0"
export PATH="$PATH:/c/VulkanSDK/1.4.309.0/Bin"
export CMAKE_PREFIX_PATH="C:/VulkanSDK/1.4.309.0/Lib;C:/VulkanSDK/1.4.309.0"
bun install
bun tauri build
```

Output: `src-tauri/target/release/bundle/nsis/Handy_<version>_x64-setup.exe` (install with
`/S` for silent) and `.../bundle/msi/Handy_<version>_x64_en-US.msi`. Stop any running
`handy.exe` first. A cold build takes about 10 minutes.

### Build tweaks carried by this fork

- `tauri.conf.json`: dropped the Windows `signCommand`, which invoked
  `trusted-signing-cli` against cjpais's Azure certificate. Without this the bundle step
  fails with `failed to bundle project: program not found`.
- The build still ends with `A public key has been found, but no private key` — that is the
  updater signature, which needs cjpais's key. Bundling has already succeeded at that point,
  so the error is harmless. It does mean auto-update is inert for this build, which is what
  we want: an upstream release cannot silently replace it and drop the feature.
