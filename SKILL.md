---
name: AgentBroPet
description: Create, repair, validate, visually QA, and package AgentBro/Codex-compatible animated pet spritesheets from concepts, brand cues, generated images, or visual references. Use when a user wants an animated app pet, mascot, or full 8x9 pet atlas with transparent unused cells, QA contact sheets, motion previews, and pet.json packaging. This skill keeps the hatch-pet deterministic atlas pipeline while discovering and using whatever image-generation skill, model, CLI, or manual external backend is available in the current agent environment.
---

# AgentBroPet

## Overview

Create an AgentBro/Codex-compatible animated pet from a concept, brand cue, company/prospect name, one or more reference images, or any combination of those inputs.

This skill is a backend-flexible version of `hatch-pet`: it keeps the deterministic scripts for prompts, layout guides, frame extraction, atlas composition, validation, visual QA, and packaging, but it does not require Codex's `$imagegen` path. Any agent can use it if it can:

- run the bundled Python scripts
- read `imagegen-jobs.json`
- generate or obtain the requested base and row-strip images
- copy selected outputs into the run folder's `decoded/` paths

User-facing inputs are optional. If the user omits a pet name, infer one from the concept, brand, company, or reference filenames; if that is not possible, choose a short friendly name. If the user omits a description, infer one from the concept or references.

## Output Contract

The deterministic pipeline targets the same app pet format used by AgentBro/Codex:

- atlas: PNG or WebP
- dimensions: `1536x1872`
- grid: 8 columns x 9 rows
- cell size: `192x208`
- background: transparent
- unused cells: fully transparent
- package: `pet.json` plus `spritesheet.webp`

The 9 animation states are fixed:

```text
idle, running-right, running-left, waving, jumping, failed, waiting, running, review
```

Keep `imagegen-jobs.json` as the manifest filename. In AgentBroPet it means "visual generation jobs"; it is not limited to Codex `$imagegen`.

## Backend Discovery

Before generating base art, row strips, or repair rows, discover the best visual generation backend available in the current environment. Prefer explicit user choice over auto-detection.

Use this order:

1. **User-named backend or model.** If the user asks for a specific image skill, model, service, or local command, use it when available.
2. **Available image-generation skill.** If the current agent environment exposes a visual generation skill or tool, use it. Examples include `$imagegen`, `$nano-banana-pro`, a browser-attached image tool, an MCP image tool, or a project-local image-generation skill. Read that skill's own instructions before use.
3. **Known local CLI/API backend.** If an image CLI is available and credentials are configured, use it only when the user has allowed that backend. For OpenAI image CLI/API, `gpt-image-2` is the normal default; use true/native transparency models such as `gpt-image-1.5` only when the user explicitly asks for or confirms that path.
4. **Manual external backend.** If no callable image backend is available, prepare the run and prompts, then ask the user or another agent/model to generate each listed job. Continue once generated PNGs are available as local paths.

Do not silently switch to a weaker, paid, remote, or credential-requiring backend. Ask first when the backend requires new credentials, costs, network setup, model downgrades, or manual user work.

For Codex specifically, `$imagegen` is a good default if present. It is not required. If using `$imagegen`, follow its installed skill at:

```text
${CODEX_HOME:-$HOME/.codex}/skills/.system/imagegen/SKILL.md
```

If using another image skill, follow that skill's storage, model, and transparency rules instead of copying `$imagegen`-specific assumptions.

## Visual Backend Contract

Every backend must satisfy the same contract:

- read the job's `prompt_file`
- attach every listed `input_images` item when the backend supports references
- generate exactly one selected image for the job
- for `base`, generate one centered full-body pet on a flat removable background or true transparent background
- for row jobs, generate one horizontal animation strip with the exact requested frame count
- avoid visible guide marks, grids, labels, text, UI, scenery, shadows, and detached effects
- return or record an absolute `selected_source` path
- let the parent agent copy that selected source into the job's `output_path`
- mark the job complete in `imagegen-jobs.json` only after the decoded copy exists

Layout guides are internal construction references. Do not show them in user-facing progress updates, do not choose them as `selected_source`, and never copy them into `decoded/`. If guide boxes, blue safety frames, center lines, or labels appear in a generated row strip, treat that row as failed and regenerate it.

Only the base job may be prompt-only. Every row-strip job must use the input images listed in `imagegen-jobs.json`, including the canonical base reference created after the base output is copied.

When a backend cannot attach references, stop and tell the user which row cannot be generated reliably. Do not fake row strips with local drawing, tiling, transforms, or code-generated placeholders.

## Brand Discovery

If the user provides only a brand, company, product, or prospect name rather than a concrete avatar description or reference image, run lightweight brand discovery before preparing the pet run. Use web search when available, preferring official sources such as the brand site, product pages, docs, about pages, press pages, or brand pages.

Skip discovery when the user already provides a concrete mascot/avatar description or reference images, unless the user explicitly asks for brand research.

Discovery output should be a compact markdown brief covering:

- identity/category
- audience/use context
- visual system
- personality/tone
- product/domain motifs
- mascot translation cues
- avoidances
- evidence/confidence

End the brief with a compact `Generation handoff` section containing exactly:

```text
brand_name=<canonical brand/product name>
brand_brief=<one sentence, max 45 words, covering palette/tone/domain motifs/personality>
avatar_seed=<short mascot-safe visual idea, no logo copying>
avoid=<short comma-separated list>
brand_sources=<comma-separated source URLs>
```

Do not copy logos, readable marks, UI screenshots, slogans, or text. Clearly label mascot guidance that is inferred rather than directly sourced.

Pass the brief path to `prepare_pet_run.py` as `--brand-discovery-file`, pass `avatar_seed` through `--pet-notes` when the user did not provide a better avatar description, and pass `brand_name`, `brand_brief`, and repeated `--brand-source` values.

If web search is unavailable and the user gave only a bare brand name, ask for brand cues before generating.

## Pet-Safe Styles

Default style is `auto`: infer the pet's style from the user's prompt and references, then preserve that style across every row. If the user names a style, honor it.

Supported style presets include:

```text
pixel, plush, clay, sticker, flat-vector, 3d-toy, painterly, brand-inspired, auto
```

Any style is acceptable when it remains pet-safe:

- compact whole-body silhouette readable inside a `192x208` cell
- consistent face, proportions, material, palette, and props across all rows
- clean removable chroma-key background or true transparent background
- details large enough to read at pet size
- no text, labels, UI, or readable logos unless the user explicitly provides approved reference art and asks for them

Non-pixel styles are first-class. Plush, clay, sticker, vector, 3D toy, painterly mascot, ink, and brand-inspired looks should be accepted when they satisfy the atlas and readability constraints.

## Transparency And Effects

Rows are processed into transparent `192x208` cells, so every generated pixel must either belong to the pet sprite or be cleanly removable background. Prefer pose, expression, and silhouette changes over decorative effects.

Allowed effects must satisfy all of these:

- state-relevant and useful for explaining the animation
- physically attached to, touching, or overlapping the pet silhouette
- inside the same frame slot as the pet
- opaque and hard-edged enough for clean extraction
- non-chroma-key colored
- small enough to remain readable at `192x208`

Avoid by default:

- wave marks, motion arcs, speed lines, action streaks, afterimages, blur, or smears
- detached stars, loose sparkles, floating punctuation/icons, falling tear drops, separated smoke clouds, or loose dust
- cast shadows, contact shadows, drop shadows, oval floor shadows, floor patches, landing marks, impact bursts, glow, halo, aura, or soft transparent effects
- text, labels, frame numbers, visible grids, guide marks, speech bubbles, UI panels, code snippets, checkerboard transparency, white backgrounds, black backgrounds, or scenery
- chroma-key-adjacent colors in the pet, prop, effects, highlights, or shadows
- stray pixels, disconnected outline bits, speckle/noise, cropped body parts, overlapping poses, or poses crossing into neighboring frame slots

State-specific guidance:

- `idle`: subtle breathing, blink, head/body bob, or tiny material sway. Must have visible micro-variation, but no waving, walking, running, jumping, working, reviewing, large gestures, item interactions, or new props.
- `waving`: show the wave through limb pose only. No wave marks, arcs, sparkles, symbols, or floating effects.
- `jumping`: show vertical motion through body position only. No shadows, dust, landing marks, impact bursts, bounce pads, or floor cues.
- `failed`: attached tears, smoke puffs, or stars are allowed if they obey the effects rules. No red X marks, floating symbols, detached smoke/stars, or separate tear droplets.
- `waiting`: show an expectant asking pose for approval, help, or user input. Keep distinct from idle and review.
- `running`: show active task work, processing, thinking, scanning, typing, or focused effort. Do not show literal foot-running, jogging, sprinting, directional travel, speed lines, dust, shadows, trails, or detached motion effects.
- `review`: show focus through lean, blink, eyes, head tilt, or paw/hand position. Avoid new props unless already part of the base identity.
- `running-right` and `running-left`: show directional drag movement through body, limb, and prop movement only. `running-right` faces/travels right; `running-left` faces/travels left. Cadence must visibly alternate across the loop.

## Visible Progress Plan

For every pet run, keep a visible checklist. Create it before starting, keep one step active at a time, and update it as each step finishes.

Use this checklist, replacing `<Pet>` with the pet's name or `your pet`:

1. Getting `<Pet>` ready.
2. Imagining `<Pet>`'s main look.
3. Picturing `<Pet>`'s poses.
4. Hatching `<Pet>`.

Only mark a step complete when the real file, image, or decision exists. If this is a repair run, start from the first relevant step instead of restarting the whole checklist.

## Default Workflow

1. Prepare a run folder and job manifest:

```bash
SKILL_DIR="${CODEX_HOME:-$HOME/.codex}/skills/agentbro-pet"
python "$SKILL_DIR/scripts/prepare_pet_run.py" \
  --pet-name "<Name>" \
  --description "<one sentence>" \
  --reference /absolute/path/to/reference.png \
  --output-dir /absolute/path/to/run \
  --pet-notes "<stable pet description>" \
  --brand-discovery-file /absolute/path/to/brand-discovery.md \
  --brand-name "<optional researched brand name>" \
  --brand-brief "<optional compact researched brand cue sentence>" \
  --brand-source "https://example.com/source" \
  --style-preset auto \
  --style-notes "<optional freeform style notes>" \
  --force
```

For text-only requests, pass the concept through `--pet-notes` and omit `--reference`. For brand-only requests, run brand discovery first.

2. Inspect `imagegen-jobs.json` for the next ready jobs:

```bash
jq '.jobs[] | {id, kind, status, depends_on, prompt_file, retry_prompt_file, input_images, output_path, derivation_policy}' /absolute/path/to/run/imagegen-jobs.json
```

A job is ready when its `status` is not `complete` and every id in `depends_on` is already complete.

3. Generate visual jobs through the selected backend:

- Generate and copy `base` first.
- Generate and copy `idle` and `running-right` next as the identity and gait check.
- Inspect `running-right`; mirror `running-left` only when visual identity, prop placement, markings, lighting, and direction semantics remain correct.
- Generate `running-left` normally when mirroring would change meaning or identity.
- Generate the remaining rows using every input image listed for each job.

For each ready visual job, use the prompt file and all listed input images with their role labels. The backend may be a skill, model API, CLI, local UI workflow, or manual external model. The parent agent records the selected source path in the manifest.

4. After selecting a generated output for a job, copy it into the decoded output path and mark the job complete. For `base`, also create the canonical identity reference:

```bash
RUN_DIR=/absolute/path/to/run
JOB_ID=<job-id>
SOURCE=/absolute/path/to/generated-output.png
OUTPUT_REL=$(jq -r --arg id "$JOB_ID" '.jobs[] | select(.id == $id) | .output_path' "$RUN_DIR/imagegen-jobs.json")
mkdir -p "$(dirname "$RUN_DIR/$OUTPUT_REL")"
cp "$SOURCE" "$RUN_DIR/$OUTPUT_REL"
```

```bash
if [ "$JOB_ID" = "base" ]; then mkdir -p "$RUN_DIR/references"; cp "$RUN_DIR/$OUTPUT_REL" "$RUN_DIR/references/canonical-base.png"; fi
```

```bash
UPDATED_AT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
TMP_MANIFEST=$(mktemp)
jq --arg id "$JOB_ID" --arg source "$SOURCE" --arg at "$UPDATED_AT" '(.jobs[] | select(.id == $id)) += {status: "complete", source_path: $source, completed_at: $at}' "$RUN_DIR/imagegen-jobs.json" > "$TMP_MANIFEST"
mv "$TMP_MANIFEST" "$RUN_DIR/imagegen-jobs.json"
```

5. Derive `running-left` only when it is visually safe:

```bash
python "$SKILL_DIR/scripts/derive_running_left_from_running_right.py" \
  --run-dir /absolute/path/to/run \
  --confirm-appropriate-mirror \
  --decision-note "<why mirroring preserves this pet's identity>"
```

That script mirrors each generated frame slot in place so the leftward row preserves the rightward row's temporal order. Do not replace it with a whole-strip mirror that reverses animation timing.

6. When all jobs are complete, run the image-processing scripts:

```bash
RUN_DIR=/absolute/path/to/run
mkdir -p "$RUN_DIR/final" "$RUN_DIR/qa"
```

```bash
python "$SKILL_DIR/scripts/extract_strip_frames.py" \
  --decoded-dir "$RUN_DIR/decoded" \
  --output-dir "$RUN_DIR/frames" \
  --states all \
  --method auto
```

```bash
python "$SKILL_DIR/scripts/inspect_frames.py" \
  --frames-root "$RUN_DIR/frames" \
  --json-out "$RUN_DIR/qa/review.json" \
  --require-components
```

```bash
python "$SKILL_DIR/scripts/compose_atlas.py" \
  --frames-root "$RUN_DIR/frames" \
  --output "$RUN_DIR/final/spritesheet.png" \
  --webp-output "$RUN_DIR/final/spritesheet.webp"
```

```bash
python "$SKILL_DIR/scripts/validate_atlas.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --json-out "$RUN_DIR/final/validation.json"
```

```bash
python "$SKILL_DIR/scripts/make_contact_sheet.py" \
  "$RUN_DIR/final/spritesheet.webp" \
  --output "$RUN_DIR/qa/contact-sheet.png"
```

```bash
python "$SKILL_DIR/scripts/render_animation_previews.py" \
  --frames-root "$RUN_DIR/frames" \
  --output-dir "$RUN_DIR/qa/previews"
```

If preview GIFs show size popping or baseline jumps caused by extraction, and the original row strip itself had stable scale and placement, rerun extraction with `--method stable-slots`, then rerun inspection, atlas composition, validation, contact sheet generation, and previews:

```bash
python "$SKILL_DIR/scripts/extract_strip_frames.py" \
  --decoded-dir "$RUN_DIR/decoded" \
  --output-dir "$RUN_DIR/frames" \
  --states all \
  --method stable-slots
```

```bash
python "$SKILL_DIR/scripts/inspect_frames.py" \
  --frames-root "$RUN_DIR/frames" \
  --json-out "$RUN_DIR/qa/review.json" \
  --require-components \
  --allow-stable-slots
```

Use `stable-slots` only as a QA-driven correction.

7. Package the output.

For Codex-compatible local pets:

```bash
RUN_DIR=/absolute/path/to/run
PET_ID=$(jq -r '.pet_id' "$RUN_DIR/pet_request.json")
DISPLAY_NAME=$(jq -r '.display_name' "$RUN_DIR/pet_request.json")
DESCRIPTION=$(jq -r '.description' "$RUN_DIR/pet_request.json")
PET_DIR="${CODEX_HOME:-$HOME/.codex}/pets/$PET_ID"
mkdir -p "$PET_DIR"
cp "$RUN_DIR/final/spritesheet.webp" "$PET_DIR/spritesheet.webp"
jq -n --arg id "$PET_ID" --arg displayName "$DISPLAY_NAME" --arg description "$DESCRIPTION" '{id: $id, displayName: $displayName, description: $description, spritesheetPath: "spritesheet.webp"}' > "$PET_DIR/pet.json"
```

For AgentBro or generic project output, put the same two files under a user-selected project folder such as:

```text
output/agentbro-pets/<pet-id>/
  pet.json
  spritesheet.webp
```

Write `qa/run-summary.json` after packaging:

```bash
jq -n --arg run_dir "$RUN_DIR" --arg spritesheet "$RUN_DIR/final/spritesheet.webp" --arg validation "$RUN_DIR/final/validation.json" --arg contact_sheet "$RUN_DIR/qa/contact-sheet.png" --arg review "$RUN_DIR/qa/review.json" --arg package "$PET_DIR" '{ok: true, run_dir: $run_dir, spritesheet: $spritesheet, validation: $validation, contact_sheet: $contact_sheet, review: $review, package: $package}' > "$RUN_DIR/qa/run-summary.json"
```

## Visual QA

After deterministic image processing, inspect `qa/contact-sheet.png` and `qa/previews/*.gif` before accepting the pet. Deterministic validation is necessary but not sufficient.

Block acceptance if any row changes species/body type, face, markings, palette, material, prop design, style, prop side unexpectedly, or overall silhouette. Motion previews must also reject unintended size popping, reversed or stagnant directional cadence, wrong facing direction, and idle loops that are technically different but visually inert.

If the environment supports visual QA workers or subagents, use them. Otherwise, inspect the contact sheet and preview GIFs directly. The QA result must cover all 9 rows.

## Repair Workflow

If frame inspection or visual QA fails, read `qa/review.json`, regenerate the smallest failing scope, copy the replacement row into the same decoded output path, and keep that job marked complete with the new `source_path` and `completed_at`. Repair the failed row, not the whole sheet.

For identity repairs, use the canonical base image, original references, contact sheet, and exact row failure note as grounding context.

For extraction-induced motion popping, do not regenerate imagery first. If the source strip already preserves row-level scale and baseline, rerun the deterministic pipeline with `--method stable-slots`, inspect with `--allow-stable-slots`, then re-check the preview GIFs. Regenerate the row only when the original strip itself is clipped, unstable, or semantically wrong.

## Rules

- Keep image generation backend-flexible; do not hard-require Codex `$imagegen`.
- Prefer auto-discovered image-generation skills/models already available to the user's current agent.
- Ask before using a backend that requires credentials, network setup, paid API calls, model downgrades, or manual user work.
- Keep reference images attached/visible whenever the chosen backend supports references.
- Attach the row's `references/layout-guides/<state>.png` image to every row-strip job when the backend supports layout references, and do not accept outputs that copy guide pixels.
- Generate every normal visual job through an image backend: base plus all row strips that are not explicitly approved `running-left` mirror derivations.
- Treat only the base job as eligible for prompt-only generation; every row job must attach its listed grounding images when supported.
- Generate `running-right` before deciding whether `running-left` can be mirrored.
- When `running-left` is mirrored, preserve frame order and timing semantics through the deterministic script.
- Do not derive or reuse `waiting`, `running`, `failed`, `review`, `jumping`, or `waving` from another state.
- Never substitute locally drawn, tiled, transformed, or code-generated row strips for missing backend outputs.
- Only mark a visual job complete after its selected output has been copied into the decoded output path.
- Use the chroma key stored in `pet_request.json`; do not force a fixed green screen.
- Treat visual identity or style drift as a blocker even when `qa/review.json` and `final/validation.json` have no errors.
- Treat `qa/review.json` errors as blockers. Warnings require visual review.

## Acceptance Criteria

- Final atlas is PNG or WebP, `1536x1872`, transparent-capable, and based on `192x208` cells.
- Used cells are non-empty and unused cells are fully transparent.
- Atlas follows the row/frame counts in `references/animation-rows.md`.
- Contact sheet and per-row motion previews have been produced and visually inspected.
- `qa/review.json` has no errors.
- Row-by-row review confirms the animation cycles are complete enough for AgentBro/Codex app use.
- Motion previews do not show unintended size popping, reversed directional cadence, or wrong row semantics.
- Non-pixel styles are accepted when readable at pet size and consistent across rows.
- `pet.json` and `spritesheet.webp` are packaged together.
