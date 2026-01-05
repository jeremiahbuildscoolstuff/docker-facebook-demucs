# AI Stem Splitter (Demucs-Based) — MVP Roadmap

## Purpose
Ship a paid, usable stem-splitting product quickly, validate willingness to pay, and build toward a future SaaS/API platform without rewriting core architecture.

## References
- Base Docker wrapper starting point: [xserrat/docker-facebook-demucs](https://github.com/xserrat/docker-facebook-demucs)
- Demucs documentation and separation options: [facebookresearch/demucs README](https://github.com/facebookresearch/demucs?tab=readme-ov-file)

## Guiding principles
- Demucs is infrastructure, not differentiation.
- Reliability and UX before scale.
- One happy path in MVP; everything else is post-MVP.
- Local-first: prove quality and the workflow before deployment complexity.
- Every stage must be runnable and testable.

## Glossary
- Stem: an isolated track (e.g., vocals, drums, bass, other).
- 2-stem: vocals + accompaniment (via Demucs `--two-stems`).
- 4-stem: vocals, drums, bass, other.
- Job: one separation request from upload to downloadable outputs.
- Worker: the component that executes Demucs and manages inputs/outputs.
- Retention: how long outputs are kept before deletion (temporary in MVP).

## Stage gates (overview)
- Stage 1: Validate separation quality/performance locally.
- Stage 2: Wrap Demucs execution behind a callable backend interface.
- Stage 3: Minimal local web UI (upload → split → download).
- Stage 4: Single-server deployment with queue + object storage.
- Stage 5: Payments (Stripe Checkout) gating execution.
- Stage 6: Launch + feedback loop with basic metrics.

## Stage 1 — Local Demucs validation (Day 1–2)
### Objective
Confirm separation quality, processing time, output format, and hardware requirements using the existing Docker-based workflow in this repo.

### Inputs
- Test audio files (MP3 and WAV), ideally 3–5 minute tracks.
- Target models to compare (at least):
  - `htdemucs`
  - `htdemucs_ft` (fine-tuned variant)
- Configs to compare:
  - 4-stem separation (default)
  - 2-stem separation (e.g., vocals-only extraction via `--two-stems`)

### How to run (current repo workflow)
- Place a file into `input/` (example: `input/mysong.mp3`)
- Run separation:
  - `make run track=mysong.mp3`
  - `make run track=mysong.mp3 model=htdemucs_ft`
  - For 2-stem (example separating vocals): `make run track=mysong.mp3 splittrack=vocals`

### Output expectations
Outputs should be deterministic and predictable (path + naming). The conceptual target layout is:

song_name/
├── vocals.wav
├── drums.wav
├── bass.wav
└── other.wav

### Acceptance criteria (done)
- Quality is acceptable relative to competitors for typical inputs.
- Performance is acceptable on at least one target machine class:
  - Consumer Nvidia GPU instance, or
  - Apple Silicon (M-series) local dev box
- Output structure and naming are consistent run-to-run.
- Clear list of “known bad inputs” (if any) is documented (very long tracks, corrupted files, exotic codecs).

### Out of scope (not now)
- Web UI
- Cloud deployment
- Accounts
- Payments

## Stage 2 — Local backend wrapper (Day 3–4)
### Objective
Turn Demucs from a CLI invocation into a callable service boundary (local only), with predictable inputs/outputs and robust failure handling.

### Deliverables
- A minimal worker/service interface that can:
  - Accept an audio file path (or uploaded file reference)
  - Accept stem configuration (2-stem vs 4-stem)
  - Spawn and supervise a Demucs job
  - Return a structured result containing output paths (or an error)
- Safety rails:
  - Timeout protection per job
  - File size limits
  - Graceful failure with actionable error messages

### Local architecture (target)
Next.js (UI + API routes)
  -> Python worker (Demucs runner)
    -> Local filesystem

### Acceptance criteria (done)
- A caller can programmatically split a file and obtain predictable outputs.
- Errors are caught and reported without crashing the service.
- The output path convention is stable and documented.

### Out of scope (not now)
- Distributed queueing
- Object storage
- GPU autoscaling

## Stage 3 — Minimal web UI (localhost) (Day 5–6)
### Objective
Create the simplest possible UX a non-technical user can understand: upload → split → download.

### UI features (MVP)
- Single page
- Drag-and-drop upload
- Stem selection: 2-stem / 4-stem
- “Split” action
- Progress indicator (at minimum: queued/running/done)
- Download links for outputs

### Acceptance criteria (done)
- A normal user can upload, wait, and download results locally.
- No crashes on normal input sizes and formats validated in Stage 1.
- Errors are presented in plain language with a retry path.

### Out of scope (not now)
- Accounts
- Advanced Demucs options surface area
- Saved libraries/history

## Stage 4 — Single-server deployment (Day 7–9)
### Objective
Deploy the MVP so it is publicly accessible at minimal cost, with basic concurrency support.

### Target infrastructure (MVP)
- One GPU-capable server (RunPod / Vast / Paperspace class)
- Dockerized services:
  - Web app
  - Worker
- Object storage (S3-compatible) for inputs/outputs
- Job queue (minimal but reliable)

### Storage rules
- Temporary files only.
- Auto-delete after 24–72 hours.
- No permanent user library.

### Acceptance criteria (done)
- Public URL works end-to-end.
- Supports 2–3 concurrent jobs with predictable cost behavior.
- Outputs are reliably retrievable via download links until retention expires.

### Out of scope (not now)
- Multi-region
- Multi-GPU autoscaling
- Advanced observability

## Stage 5 — Payments + monetization (Day 10–12)
### Objective
Validate willingness to pay using the simplest payment flow that gates compute cost.

### Payments (MVP)
- Stripe Checkout
- Pay before processing
- One-time purchases only

### Pricing (MVP hypothesis)
- 2 stems: $3–5
- 4 stems: $5–8

### Enforcement
- Payment required to start a job.
- Rate limit unpaid users.
- Optional: tiny demo file size/time limit (only if it doesn’t distract from paid flow).

### Acceptance criteria (done)
- At least one real payment processed successfully.
- No payment-related crashes that strand users without results.
- Receipts and download links are clear and reliable.

### Out of scope (not now)
- Subscriptions
- Coupons/discount campaigns
- Team accounts

## Stage 6 — MVP launch + feedback loop (Day 13–14)
### Objective
Validate market demand and learn what breaks.

### Launch deliverables
- Simple landing page (clear promise + CTA)
- Minimal example use cases:
  - Remixing
  - Karaoke
  - DJ edits

### Metrics to track (MVP)
- Conversion rate (visit → purchase)
- Average file length
- Time to completion
- Repeat usage
- Drop-off points (upload, pay, processing, download)

### Acceptance criteria (done)
- People pay.
- Jobs complete reliably.
- Quality meets expectations for the tested input set.
- Infrastructure cost stays below revenue (or is on a clear path to).

## Post-MVP footnotes (do not build yet)
### Product evolution
- DAW export presets
- Batch uploads
- Stem recombination
- Key/BPM detection
- Vocal isolation tuning

### Platform evolution
- Developer API (usage-based)
- Webhooks
- SDKs
- White-label licensing

### Infrastructure evolution
- Multi-GPU scaling
- Model switching (Demucs and other separators)
- Client-side processing (WebGPU) for eligible devices

### Business evolution
- Subscription tiers
- Team plans
- Education licensing
- Partner integrations

## Operating rule
Do not over-engineer before the first dollar. Keep boundaries clean so models, pricing, and infrastructure can change without a rewrite.


