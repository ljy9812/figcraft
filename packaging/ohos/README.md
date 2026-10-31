# FigCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `figcraft-web` build — the same static site published as
`figcraft-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from
the HAP's own `rawfile`: no network origin, no server-side code.

The CJK (Chinese) variant is a build-time env switch — `FIGCRAFT_EMBED_CJK=1` — over the official code; see the recipe below.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.figcraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks — attached (document listener) and programmatic/detached (`HTMLAnchorElement.prototype.click` override) — are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` (with a filename field) |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### CJK (Chinese) variant — in this branch

no source change — the official code embeds its CJK faces when FIGCRAFT_EMBED_CJK=1 is set at build time (faces from the craft-fonts checkout); add that env var to the recipe below.

Self-build recipe (Windows host, Rust stable, `trunk` 0.21.x; pin versions
per the upstream repo — on networks where the wasm-opt download hangs,
build with the optimizer off or assemble the .stage outputs by hand):

```powershell
git clone -b hans-wqy-microhei https://github.com/ljy9812/craft-fonts.git craft-fonts-cjk
$env:CRAFT_FONTS_DIR = "<repo>\craft-fonts-cjk"      # absolute path
cd apps\figcraft-web
trunk build --release
# deploy dist\web\ into packaging\ohos\entry\src\main\resources\rawfile\web\:
# swap the wasm+glue pair, copy snippets\ and fonts\ when the build produced
# them, and rewrite index.html's asset hash — a stale hash (or a stale SRI
# integrity attribute) double-fetches the wasm behind a white window.
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.figcraft.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23, physical keyboard +
touchscreen): install & launch and zh-Hans UI rendering with the
self-built CJK wasm (FIGCRAFT_EMBED_CJK=1). Deeper interaction criteria
in progress.

Known limitations:

- Canvas text layers render CJK as tofu although the UI face renders — the
  same application-layer font-routing family as designcraft.
- Code export from the Inspect panel is clipboard-only (by design, no file
  write path).
