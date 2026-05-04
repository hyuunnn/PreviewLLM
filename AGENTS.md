# TranslatePanel

macOS menu-bar utility that captures screen content or selected text via global
hotkeys, runs OCR through Apple Vision, and streams the result to a CLI-based
LLM (`claude`, `codex`, `gemini`, `lms`, `apfel`, `copilot`) shown in a
floating panel.

## Build

```bash
./build.sh                       # release .app bundle in build/
open build/TranslatePanel.app
swift build                      # debug build only
```

SPM project, macOS 14+, Swift 5.10. Single executable target. No external Swift
dependencies. Bundle ID: `com.translate.panel`. Runs as `LSUIElement` (no Dock
icon). Requires Accessibility and Screen Recording permissions.

## Hotkeys

| Shortcut       | Action                                       |
|----------------|----------------------------------------------|
| `Cmd+Shift+\`  | Toggle floating panel                        |
| `Cmd+Shift+,`  | Translate selected text (AX API → clipboard) |
| `Cmd+Shift+.`  | Full-screen capture + OCR + translate        |
| `Cmd+Shift+'`  | Region capture + OCR + translate             |

Registered via Carbon Event Manager in `HotkeyManager`, not SwiftUI shortcuts.

## Architecture

- **MVVM**: `ChatView` observes `ChatViewModel` (`@StateObject` / `@Published`).
- **LLM calls are process-based, not network**: `Process()` spawns a CLI
  binary; stdout is streamed via `readabilityHandler`.
- **CLI binary paths** are resolved once with `zsh -li -c "which <binary>"`
  and cached.
- **Translation prompts are hardcoded in Korean** inside
  `ChatViewModel.sendWithAction()`.
- **OCR** via `VNRecognizeTextRequest` (en, ko, ja, zh-Hans, zh-Hant).
- **Screen capture** via `ScreenCaptureKit`; the app's own windows are
  excluded from captures.
- **Region selection** in `RegionCaptureView` converts Cocoa (origin
  bottom-left) to Core Graphics (origin top-left) via
  `primaryHeight - origin.y - height` — `ScreenCaptureKit` requires CG
  coordinates while `NSEvent` delivers Cocoa.
- Settings persist in `UserDefaults` via `@AppStorage`.

## Adding a New LLM Provider

1. Create a struct conforming to `LLMProvider` in `Sources/LLMProvider.swift`.
2. Implement `buildArguments()` and `formatPrompt()`.
3. Register the instance in `LLMProviderRegistry.all`.
4. Add `settings.modelPlaceholder.<id>` to both
   `Resources/{en,ko}.lproj/Localizable.strings`.

`apfel` and `lms` pass the prompt as a CLI argument (not stdin) — set
`passesPromptViaArgument`.

## Localization

`Resources/{en,ko}.lproj/Localizable.strings`. Keys must stay in sync between
both files (a missing key renders as the raw key string). Korean is the
default localization. Loaded via `L()` in `Sources/Localization.swift`.

## Testing

No automated test target. Verify manually after changes — build, run, exercise
all four hotkeys and any provider you touched.
