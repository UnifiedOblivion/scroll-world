# Pipeline: OpenArt recipes + copy-paste local scripts (bash 3.2 safe)

Generation runs through the OpenArt MCP tools (you drive them as tool calls — there is
no generation CLI). Everything local — frame extraction, encoding, posters, the SSIM
gate — is bash + ffmpeg, copy-paste below.

Set these once. `NAMES` is the ordered section ids; the last is the hero/finale.

```bash
WORK=/tmp/scroll-world           # scratch dir for prompts, sources, frames
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order
```

**Chain video model — ONE for every chained clip** (SKILL Step 4 roster). Its
`image2video` form must take `startFrame`, and `endFrame` too for connectors — verify
with `openart_model_form_get(model, "image2video")`. Default `kling-3-omni`; param
blocks below. Reference-only (element) models can't hold a seam.

```
KLING FINAL (dive/leg):  { "resolution": "pro", "duration": 8,  "generateSound": false,
                           "multiShot": false, "videoCount": 1 }
KLING FINAL (connector): { "resolution": "pro", "duration": 5,  "generateSound": false,
                           "multiShot": false, "videoCount": 1 }
KLING DRAFT tier:        same blocks with "resolution": "std"   (~70% of pro cost —
                           worth a draft pass only before 4k finals; SKILL Step 4)
KLING GOLD tier:         same blocks with "resolution": "4k"    (quote it first — ~3.4× pro)
```

Always: `openart_model_form_get` before the first job of a batch (fields drift),
`openart_model_cost` with the exact params before spending (prices drift; the numbers
in SKILL Step 1 are 2026-07 quotes).

**Resume / idempotency.** The output file *is* the run state: before generating any
asset, check whether its local file already exists and is non-empty — if so, skip the
job. A crash, credit stall, or filter re-roll never costs finished assets: re-run the
same loop and only the missing pieces regenerate. To force a re-roll of one asset,
delete its file first (`rm "$WORK/dive_shop.mp4"`, then regenerate just that one).
Check where a run stands any time:

```bash
status() { for n in $NAMES; do
  printf '%-10s still:%s dive:%s\n' "$n" \
    "$([ -s "$WORK/still_$n.png" ] && echo ok || echo -- )" \
    "$([ -s "$WORK/dive_$n.mp4" ] && echo ok || echo -- )"
done; ls "$WORK"/conn_*.mp4 2>/dev/null | while read f; do printf 'conn: %s ok\n' "$f"; done; }
```

## 0. The two OpenArt lanes you'll repeat all run

**Generate → wait → download.** `openart_generate_image` / `openart_generate_video`
returns a `historyId` (status PENDING). Poll `openart_creation_wait(historyId)` — video
often outlasts one wait window; on STILL_RUNNING call it again with the same historyId.
When finished, curl the result's media URL down to its `$WORK` file:

```bash
curl -fsSL "<result media url>" -o "$WORK/dive_farm.mp4"
```

**Upload a local frame → reference object.** Chained clips are seeded with locally
extracted frames, which must be uploaded before generating:

1. `openart_upload_sign` with `mediaType: "image"`, the exact byte `size`
   (`wc -c < file`), `contentType: "image/png"`, `purpose: "create-video"`, and a
   meaningful `label` (e.g. `"leg-2 last frame"`).
2. PUT the bytes to the returned `signURL`:
   ```bash
   curl -fsS -X PUT -H "Content-Type: image/png" --data-binary "@$WORK/last_farm.png" "<signURL>"
   ```
3. Use the returned reference object (`{type:"image", id, url, label}`) as the job's
   `startFrame` / `endFrame`.

Uploads are reusable within the run — don't re-upload the same frame for a re-roll.

## 1. Scene stills (Step 2) — anchor first, then batch

Write one prompt file per section to `$WORK/still_<name>.txt` (see prompts.md).
Image model: **Seedream 5.0 Pro**, resolved from `openart_model_list` by display name
(SKILL Step 0 availability gate — never infer a slug, never silently downgrade).
Aspect **16:9** (Kling `image2video` inherits the start frame's aspect), 2K-class
quality per the current form.

**Do NOT batch all N immediately.** Generate ONE anchor still first (the most
representative scene) via `text2image`, download it to `$WORK/still_<anchor>.png`, and
get the user's approval on the art direction. A style miss caught on the anchor costs
1 gen; caught after the batch it costs N.

Then batch the remaining N-1 **in `image2image` mode with the approved anchor as the
reference image** to lock the style (fetch the `image2image` form for the exact
reference field — it drifts between models). Skip any still whose
`$WORK/still_<name>.png` already exists. Download each result as it finishes.

Convert to webp for the site (and optionally run knockout.py first for transparency):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

Review the batch for cohesion before continuing. Re-roll any off-style one
(`rm "$WORK/still_shop.png"`, regenerate just that one — the anchor style lock stays
in force).

## 2. Dive-in / leg clips (Step 4)

Prompt files at `$WORK/dive_<name>.txt`. Skip any name whose `$WORK/dive_<name>.mp4`
exists.

- **Architecture B (dives):** upload each solid-bg still (§0 upload lane), then per
  scene: `openart_generate_video(model: "kling-3-omni", mode: "image2video", params:
  {prompt, startFrame: <uploaded still ref>, ...KLING FINAL dive block})`. Jobs can run
  concurrently — submit the batch, then wait on each historyId.
- **Architecture A (legs):** strictly sequential. Leg 0 starts from scene-0's uploaded
  still; each later leg's `startFrame` is the **previous leg's extracted last frame**
  (§3, uploaded via §0). No `endFrame`. Eyeball each leg's last frame before chaining
  the next (SKILL Step 4 camera grammar).

Download every result to `$WORK/dive_<name>.mp4`. Re-roll individual failures
(transient errors and filter flags are per-job — never restart the batch).

## 3. Extract boundary frames — the seam handoff (Step 5)

For each adjacent pair, the connector's start = dive_i's LAST frame, end = dive_{i+1}'s
FIRST frame — extracted from the **rendered videos**, never the stills.

```bash
for n in $NAMES; do
  ffmpeg -v error -ss 0 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/first_$n.png"      # establishing
  ffmpeg -v error -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/last_$n.png" # interior
done
```

## 4. Connector clips (Step 5)

Prompt files at `$WORK/conn_<i>.txt` (i = 1..N-1). Skip any i whose
`$WORK/conn_<i>.mp4` exists. For each adjacent pair (prev, next):

1. Upload `$WORK/last_<prev>.png` and `$WORK/first_<next>.png` (§0 upload lane) — skip
   uploads already made this run.
2. `openart_generate_video(model: "kling-3-omni", mode: "image2video", params: {prompt,
   startFrame: <last_<prev> ref>, endFrame: <first_<next> ref>, ...KLING FINAL
   connector block})`.
3. Wait, download to `$WORK/conn_<i>.mp4`.

Connectors need `endFrame` — any SKILL Step 4 roster model accepts it; element-reference
modes never do.

## 5. Encode everything for scrubbing (Step 6)

Native resolution — encode whatever ffprobe reports for the downloaded render (kling
`std` is 720-class, `pro` 1080-class; **never upscale**), crf 20, GOP 8, light sharpen,
no audio, faststart. Same for dives + connectors.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); enc "$f" "$ASSETS/vid/conn$i.mp4"; done
```

Now the engine config's `sections[k].clip = assets/vid/<name>.mp4` and
`connectors = [assets/vid/conn1.mp4, …]` (length N-1, in order).

## 5b. Posters — extract from the ENCODED clips (kills the still→video pop)

The clip is a *re-render* of the still, so if the still is the loading poster, the
moment the video paints there's a visible render-drift jump — on the very first scene a
visitor sees. Same doctrine as the connectors: hand off actual frames. The poster must
be the encoded clip's own first frame:

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0 -i "$ASSETS/vid/$n.mp4" -frames:v 1 -q:v 2 "$WORK/poster_$n.png"
  cwebp -quiet -q 84 "$WORK/poster_$n.png" -o "$ASSETS/$n-poster.webp"
done
```

Wire as `sections[k].poster = 'assets/<name>-poster.webp'`. Keep `still` too — it stays
the reduced-motion artwork and the no-clip fallback (the engine prefers `poster` while a
clip will load, `still` otherwise).

## 5c. Verify the seams — automated, before any eyeballing

Seamlessness is the product; don't ship it on a squint. Every seam in the chain must be
near-identical across its boundary frames. SSIM-check them all from the encoded files
(chain order: dive0, conn1, dive1, conn2, … — for architecture A it's just leg0, leg1, …):

```bash
# last frame of A vs first frame of B, SSIM score on stdout
seam_ssim() { # fileA fileB
  ffmpeg -v error -y -sseof -0.05 -i "$1" -frames:v 1 "$WORK/_sa.png"
  ffmpeg -v error -y -ss 0      -i "$2" -frames:v 1 "$WORK/_sb.png"
  ffmpeg -v info -i "$WORK/_sa.png" -i "$WORK/_sb.png" -lavfi ssim -f null - 2>&1 \
    | grep -o 'All:[0-9.]*' | cut -d: -f2
}

check() { # fileA fileB label
  s=$(seam_ssim "$1" "$2")
  case $(awk -v s="${s:-0}" 'BEGIN{ if (s>=0.90) print "pass"; else if (s>=0.75) print "warn"; else print "fail" }') in
    pass) echo "PASS  $3  ssim=$s" ;;
    warn) echo "WARN  $3  ssim=$s (crossfade will mostly hide it — eyeball this seam)" ;;
    *)    echo "FAIL  $3  ssim=$s — endpoints are NOT the neighbours' frames; redo this connector (SKILL Step 5)" ;;
  esac
}

# Architecture B (dive/conn interleave):
set -- $NAMES ; i=0 ; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1))
    check "$ASSETS/vid/$prev.mp4" "$ASSETS/vid/conn$i.mp4" "$prev>conn$i"
    check "$ASSETS/vid/conn$i.mp4" "$ASSETS/vid/$n.mp4"    "conn$i>$n"
  fi ; prev="$n"
done
# Architecture A (sequential legs): check "$ASSETS/vid/legI.mp4" "$ASSETS/vid/legI+1.mp4" pairs.
```

Thresholds from the frame-handoff physics: a true actual-frame handoff scores ≥0.95
even after encoding; ≥0.90 pass, 0.75–0.90 warn (the endFrame-conditioned render landed
close but not exact — the engine crossfade usually covers it), <0.75 means a still was
used as an endpoint or the wrong frame was extracted — regenerate, don't rationalize.
Run this after every re-roll too: replacing one clip can silently break BOTH of its
seams.

## 6. Mobile encodes (Step 6) — only if the user picked a mobile tier

**Skip this section if the user chose the crop-safe tier in the Step 1
interview.** Scrubbing sets `currentTime` every frame, and a phone decoder's **seek cost scales with
how many frames it must decode from the nearest keyframe** — so a `-g 8` master
that scrubs fine on a laptop stutters on a phone. A **smaller frame + tighter GOP** fixes
that (and halves the bytes on cellular). Produce a `-m.mp4` sibling for every clip:

```bash
# 720p, GOP 4 (twice the keyframes = ~half the seek-decode work), crf 23, same sharpen/faststart.
encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=-2:720,unsharp=5:5:0.6:5:5:0.0" \
  -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
  -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; echo "encm $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do encm "$WORK/dive_$n.mp4" "$ASSETS/vid/$n-m.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); encm "$f" "$ASSETS/vid/conn$i-m.mp4"; done
```

Extract each mobile encode's first frame as its poster (§5b doctrine — the poster must
match the encode the device actually gets):

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0 -i "$ASSETS/vid/$n-m.mp4" -frames:v 1 -q:v 2 "$WORK/poster_${n}_m.png"
  cwebp -quiet -q 84 "$WORK/poster_${n}_m.png" -o "$ASSETS/$n-poster-m.webp"
done
```

Wire the variants in the engine config — the engine serves them on phone-class devices
(screen short side ≤600 CSS px; tablets/iPads get the master), falling back to the
desktop `clip` when a mobile one is absent:

```js
sections[k].clipMobile   = 'assets/vid/<name>-m.mp4';
sections[k].posterMobile = 'assets/<name>-poster-m.webp';
connectorsMobile = ['assets/vid/conn1-m.mp4', …];   // length N-1, in order
```

If phone scrubbing still stutters, tighten the GOP further (`-g 2`, or `-g 1` for all-intra
= instant seeks at the cost of larger files); if cellular weight is the bigger worry, raise
`crf` (24–26) or drop to `scale=-2:600`. If the master is already 720-class (kling `std`),
the mobile encode still pays off — the tighter GOP is what makes phone seeks cheap.
Plain mobile-tier encodes stay 16:9 — the engine centre-crops them; for true portrait
assets see §7.

## 7. Portrait tiers (Step 1.6 hero-reframe / full portrait chain) — extra credits

The gold standard for mobile is a **differently-framed render, not a crop** (Apple ships
re-art-directed per-breakpoint assets). Two levels:

**Hero reframe (cheap).** For the 1–2 scenes whose focal subject can't hold a centre
crop (usually hero + finale): regenerate JUST those stills at 9:16 (same prompt + anchor
style reference, recompose vertically), render a 9:16 dive from each (the clip inherits
the still's portrait aspect), encode with `encm()` settings, wire as that scene's
`clipMobile`/`posterMobile`. Other scenes keep the cropped 16:9 mobile encode. Seams: a
portrait dive still hands off within its own clip only, so nothing about the 16:9 chain
changes — the phone crossfades between a cropped connector and the portrait dive; keep
the reframed composition centred on the same focal point so the transition reads.

**Full portrait chain (gold, ≈2× video credits).** A complete parallel 9:16 chain:

- 9:16 stills (or reuse the 16:9 stills as `image2image` style references and prompt the
  vertical recomposition), then the full Step 4/5 flow seeded from portrait frames — own
  frame extractions, own connectors, own handoffs. **Aspect ratios cannot mix
  mid-chain**: a 9:16 clip can't continue a 16:9 frame, so the portrait chain is
  generated end-to-end as its own world.
- Run the §5c SSIM gate on the portrait chain separately (its own seam list).
- Encode with `encm()` (already 720-class; portrait at `scale=720:-2`), extract portrait
  posters, wire ALL of it as `clipMobile`/`connectorsMobile`/`posterMobile`.
- Same-model rule, same filter re-roll budget — everything from the 16:9 chain applies.

State the credit cost to the user before starting either tier (SKILL Step 1.6).

## Notes

- **Content-filter fallback across models**: if one clip keeps getting flagged on Kling
  after re-rolls + prompt scrubbing, regenerate just that clip on `byte-plus-seedance-2`
  (`image2video`, `generateAudio: false`, same uploaded start/end frames) — then return
  to the chain model. See SKILL Gotchas for the trade-off.
- **Draft tier**: same model at `resolution: "std"` — see the setup block. Don't reach
  for element-reference models for drafts: without startFrame/endFrame they can't hold a
  seam, so their output can't be chained (SKILL Step 4 rule).
- If a whole batch stalls, check `openart_account_get` for credits and each job's
  `openart_creation_get(historyId)` for the failure reason.
- Concurrency: a handful of concurrent video jobs is fine (architecture B dives,
  connectors); architecture A legs are inherently sequential. Re-roll individual
  failures rather than restarting a batch.
