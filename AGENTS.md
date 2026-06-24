# Repository Guidelines

## Project Context
This project is the uni-app X / UTS migration of the Tovigor app. The main target is Android App in HBuilderX.

- Treat `tovigor_yei_unIappX/` as independent from the legacy `tovigor_yei/` project.
- Use the legacy project as behavior and visual reference when requested, but implement with uni-app X / UTS-compatible APIs and components.
- Do not assume ordinary Vue WebView behavior applies to uni-app X App rendering.

## Development Workflow
- Use HBuilderX as the primary build and run environment.
- Do not treat `npm` scripts as the standard workflow unless the user explicitly asks.
- Prefer narrow, page-local changes unless shared behavior is clearly required.
- For UI or video behavior, validate manually on Android App or an appropriate custom base when possible.

## UTS / Script Setup Rules
- In `<script setup lang="uts">`, declare helper functions before any `computed`, `watch`, top-level constants, callbacks, or other function bodies that reference them. Do not rely on JavaScript function hoisting.
- Before finishing UTS edits, check newly added or moved helper functions are defined above their first call site, especially page handlers such as `showPanel`, `hidePanel`, `openModal`, and video helpers like `playBackgroundVideo` / `pauseBackgroundVideo`.
- Treat nullable values strictly. If an API can return `T?`, check `value != null` before calling methods or accessing properties.
- Avoid optional chaining as a substitute for understanding nullable API behavior when later method calls have side effects.
- Keep event handler parameter types explicit when the type is known, for example `UniSliderChangeEvent` or `UniVideoErrorEvent`.
- Prefer simple named functions over complex inline expressions in templates when UTS type inference becomes fragile.
- When reading DOM layout in uni-app X, do not rely on inferred return types from `getBoundingClientRect()` or `getBoundingClientRectAsync()`. Explicitly type the returned value as `DOMRect` before accessing `left`, `top`, `width`, `height`, `x`, or `y`; otherwise the UTS compiler may report field names as missing identifiers, for example "找不到名称 x/y/width/height".

## uni-app X Video Rules
- App video playback is not reliable with `autoplay` alone. Give important videos a stable `id` and use `uni.createVideoContext(id)` to call `play()`, `pause()`, and `seek()` explicitly.
- Always null-check the `VideoContext` returned by `uni.createVideoContext`.
- For looping background videos, keep `loop` but also handle `@ended` and manually `seek(0)` then `play()` as a fallback.
- For full-screen videos, use an explicit transparent tap layer above the video instead of relying on the video element to receive page gestures.
- Log full `event.detail` in `@error` handlers so runtime codec, base, or path issues can be diagnosed.
- Android x86 standard bases may not include all video compatibility libraries. Prefer real arm64 devices or a custom base with the required ABI when debugging video black screens.

## Layout Rules
- For full-screen App pages, prefer sizing from `uni.getSystemInfoSync().windowWidth/windowHeight` when video, overlays, or native components need exact full-screen coverage.
- Keep video, tap layer, masks, and control overlays aligned to the same dimensions and layer order.
- Be careful with `position: fixed` and `position: absolute`; verify App behavior rather than assuming H5/WebView semantics.
- For bottom controls, reserve content padding or use explicit overlay spacing so controls do not cover important content.

## Serial / Hardware Rules
- Preserve the `utils/serialRuntime.uts` flow for uni-app X pages unless the user explicitly asks to replace it.
- When leaving, hiding, or destroying a training page, stop force output and remove frame/status listeners.
- Do not regress safety behavior for `stopForce`, `onFrame`, `offFrame`, `onStatus`, and `offStatus`.
- If a page starts force output, every navigation path out of that page must stop force output first.

## Encoding Rules
- Treat all Chinese text as UTF-8.
- On Windows PowerShell, read Chinese files with `Get-Content -Encoding UTF8`.
- Do not edit garbled Chinese blindly. Confirm or replace with readable UTF-8 text while making related changes.
