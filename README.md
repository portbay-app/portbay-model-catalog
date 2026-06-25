# portbay-model-catalog

The signed, live-updatable catalog of on-device models (speech-to-text,
text-to-speech, image generation) that the PortBay desktop app downloads and
runs. Publishing a new `model-catalog.json` here makes new models available to
every installed app **without an app release**.

## How the app consumes this

`src-tauri/src/commands/model_catalog.rs` in the main app fetches:

- `https://github.com/portbay-app/portbay-model-catalog/releases/latest/download/model-catalog.json`
- `…/model-catalog.json.sig`

It **verifies the minisign signature against the updater public key before
parsing** (the same key/flow as app auto-updates). On any failure — network,
bad signature, schema too new — it falls back to its disk cache, then to the
`default-model-catalog.json` bundled in the app. So a broken or forged manifest
can never push unverified models; the worst case is "no newer models than what
shipped." Fresh catalogs are picked up within a 24 h cache TTL, or instantly via
the AI page's refresh.

The bundled `default-model-catalog.json` in the app is the **offline floor**;
the live manifest is merged over it (live wins per `id`, bundled fills gaps).

## Schema

`schemaVersion: 1`, then `stt[]`, `tts[]`, `image[]`. Entry shapes mirror the
Rust structs (`SttCatalogModel`, `TtsCatalogModel`, `ImageCatalogModel`). Every
entry needs at least `id`, `engine`, `repoModel`. `engine` must map to an engine
the app's sidecars actually run (STT: `whisper`/`parakeet`/`parakeet-eou`/`qwen3`;
image: `sd`/`sdxl`/`stable-diffusion`/`flux`; TTS: `kokoro`). `repoModel` is the
Hugging Face repo the sidecar downloads from. Optional `contentDigest` pins the
weights with a sha256 the sidecar re-checks at install time (supply-chain pin).

## Publishing

`model-catalog.json` is signed and published by the **Publish model catalog**
GitHub Action (`.github/workflows/publish-catalog.yml`) on push to `main` or
manual dispatch.

**Required repo secrets** (the PortBay updater signing key — the same one used
to sign app updates):

- `TAURI_SIGNING_PRIVATE_KEY`
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` (empty if the key has no password)

The Action validates the JSON, runs `tauri signer sign`, and publishes the
manifest + `.sig` to the rolling `catalog-latest` release.

## Roadmap (see the app's assessment doc)

- **Phase 0 (this repo):** activate the signed channel with today's list.
- **Phase 1:** a scheduled ingestion job queries the Hugging Face Hub API per
  modality, filters by engine compatibility + license + a curation policy
  (auto-admit), enriches metadata, computes `contentDigest`, and regenerates
  this `model-catalog.json` — so new STT/TTS/image models flow in automatically.
- **Phase 2+:** embeddings as a first-class category; an HF-direct LLM
  supplement (GGUF/MLX models not on Ollama); auto vendor/logo metadata;
  trending/newest surfacing.

Curation, enrichment, and ingestion logic will live under `scripts/`.
