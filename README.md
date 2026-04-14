# pocketai-cdn

Static CDN serving model catalogs for PocketAI apps. Hosted via GitHub Pages at `cdn.pocketaihub.com`.

## Base URL

```
https://cdn.pocketaihub.com
```

## Endpoints

### Language Models

```
GET https://cdn.pocketaihub.com/catalog.json
```

Returns the full language model catalog. Each entry includes model ID, name, provider, size, RAM/VRAM requirements, download URL, supported platforms, capabilities (e.g. vision), and context length.

### Image Models

```
GET https://cdn.pocketaihub.com/image-catalog.json
```

Returns the image generation model catalog (Stable Diffusion, FLUX, SDXL, Chroma, Z-Image, Ovis-Image, Qwen-Image). Each entry includes architecture, supported resolutions, default steps/CFG, and auxiliary model URLs (CLIP, T5, VAE).

### Video Models

```
GET https://cdn.pocketaihub.com/video-catalog.json
```

Returns the video generation model catalog (Wan 2.1/2.2 family). Each entry includes mode (t2v, i2v, flf2v), supported resolutions, frame counts, FPS, and auxiliary model URLs (T5, VAE, CLIP Vision).

### Voice Models

```
GET https://cdn.pocketaihub.com/voice-catalog.json
```

Returns the voice model catalog covering:
- **STT** - Whisper models (Tiny through Large v3 Turbo)
- **TTS** - Kokoro and Piper text-to-speech models
- **VAD** - Silero voice activity detection

### Gated Models

```
GET https://cdn.pocketaihub.com/gated-catalog.json
```

Returns gated/uncensored models separated from the main catalogs. Structured as `{ "image": [...], "video": [...], "language": [...] }`. Only fetched by non-app-store builds — app store builds never request this endpoint.

### Combined Manifest

```
GET https://cdn.pocketaihub.com/manifest.json
```

Returns a single combined manifest containing all catalogs with a top-level `version` field.

### Catalog Versions

```
GET https://cdn.pocketaihub.com/catalog-versions.json
```

Returns the current version number for each catalog category. Used by apps to check for updates without downloading the full catalogs.

```json
{
  "language": 1,
  "image": 1,
  "video": 1,
  "voice": 1
}
```

## Model Weight Tiers

Each language model entry includes a `weight` field indicating its download size class:

| Weight | Size Range | Typical Use |
|--------|-----------|-------------|
| `tiny` | < 1 GB | Older phones, fast downloads |
| `light` | 1 - 3 GB | Most phones and tablets |
| `medium` | 3 - 10 GB | Flagship phones, all desktops |
| `large` | 10 - 25 GB | Desktop / laptop only |
| `heavy` | 25 - 80 GB | High-memory workstations |
| `massive` | 80 GB+ | Multi-GPU / high-end setups |

## Where Catalogs Are Stored

There are two copies of the model catalogs that must stay in sync:

| Location | Description |
|----------|-------------|
| `pocketai-cdn/catalog.json` | **CDN (source of truth)** - language model catalog served at `cdn.pocketaihub.com/catalog.json` |
| `pocketai-cdn/image-catalog.json` | CDN - image generation model catalog |
| `pocketai-cdn/video-catalog.json` | CDN - video generation model catalog |
| `pocketai-cdn/voice-catalog.json` | CDN - voice model catalog (STT, TTS, VAD) |
| `pocketai-cdn/gated-catalog.json` | CDN - gated/uncensored models (non-app-store builds only) |
| `pocketai-cdn/manifest.json` | CDN - combined manifest (all catalogs + version) |
| `pocketai-cdn/catalog-versions.json` | CDN - version numbers for update checks |
| `local-language-model/packages/core/src/catalog-manifest.json` | **Local bundled copy** - embedded in the app's core package. Structure: `{ "version": 1, "models": [...] }` wrapping the same data as `catalog.json` |

When adding or updating models, update `catalog.json` first, then sync `manifest.json` and the local `catalog-manifest.json` to match.

## App Store vs Website Builds

Gated models (`gated-catalog.json`) are **completely stripped** from app store binaries at compile time. The defaults are all "website" (gated content visible) — flip the flag only when building for app stores.

### Desktop (Next.js / Tauri)

```bash
# Website build (default) — gated content included
npm run build

# App store build — gated content stripped from JS bundle
NEXT_PUBLIC_FOR_APP_STORES=true npm run build
```

Config: `apps/app/app/app-config.ts` reads `process.env.NEXT_PUBLIC_FOR_APP_STORES`. Next.js inlines this at build time, so the minifier removes the dead branch and the `gated-catalog.json` URL never appears in the bundle.

### Android

```bash
# Website build (default)
./gradlew assembleRelease

# App store build — R8 strips dead branch in release builds
./gradlew assembleRelease -PFOR_APP_STORES=true
```

Or set in `gradle.properties`:
```properties
FOR_APP_STORES=true
```

Config: `app/build.gradle.kts` sets `BuildConfig.FOR_APP_STORES`. R8 (`isMinifyEnabled = true`) eliminates dead code in release builds.

### iOS

```
Website build (default):  No extra flags needed.
App store build:          Xcode → Target → Build Settings →
                          "Active Compilation Conditions" → add FOR_APP_STORES
```

Config: `PocketAI/Services/AppConfig.swift` uses `#if FOR_APP_STORES` compiler directives. Code inside `#if !FOR_APP_STORES ... #endif` is not compiled into the binary at all.

## Usage in Apps

The desktop/web app references the CDN base URL in `apps/app/app/api-client.ts`:

```ts
const CDN_URL = process.env.NEXT_PUBLIC_CDN_URL || "https://cdn.pocketaihub.com";
```
