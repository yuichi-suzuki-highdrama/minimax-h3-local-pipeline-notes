# MiniMax-H3 local pipeline notes (RTX 5090)

Personal lab notes for a **ComfyUI + MiniMax-H3 (Ref2VA)** video pipeline on a single **RTX 5090**.
Timings are wall-clock on one machine; treat them as relative, not absolute benchmarks.

**Privacy:** this write-up intentionally omits local usernames, absolute home paths, character reference assets, and project codenames.

**Abbreviations used below**

| Term | Meaning |
|---|---|
| Ref2VA / R2V | MiniMax-H3 reference-to-video(+audio) mode: reference images/videos + text → video |
| PDD Acc | alibaba-pai's Parallel-Decoding-Distillation acceleration LoRA (8-step) for H3 |
| TE | the Qwen3-VL text encoder that turns prompt + refs into conditioning |
| cond | ComfyUI CONDITIONING (TE output); "768-cond" = conditioning encoded at 768p |
| uponly | latent upscale only (the H3 latent upscaler model), with no sampling in that step |
| R2V4 | lightx2v's Ref2V turbo 4-step LoRA (`minimax_h3_ref2v_turbo_4step_v0.1`) used as the refine distill |
| LX8 | lightx2v's Ref2V turbo 8-step LoRA (`minimax_h3_ref2v_turbo_8step_v1.0_768p`) |
| VSR | NVIDIA RTX Video Super Resolution (nvidia-vfx SDK) |

## Environment (all numbers below were measured here)

| Item | Value |
|---|---|
| GPU | NVIDIA RTX 5090, 32 GB VRAM (Blackwell) |
| Host | Windows 11, 128 GB RAM |
| ComfyUI | 0.34.x (master, early Sept 2026) |
| PyTorch | 2.11 + CUDA 13.0 |
| Attention | comfy-kitchen 0.2.33 (`--use-ck-attention`) |
| H3 weights | ref2va pruned int8 convrot UNET, Qwen3-VL-32B int8 convrot text encoder |

## Stack

- ComfyUI + MiniMax-H3 Ref2VA (pruned int8 convrot UNET works)
- Attention: **comfy-kitchen** (`--use-ck-attention`) — dense kitchen path is the locked mainline
- alibaba-pai's official **PDD Acc 8-step LoRA** ([model card](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs)), loaded through the community node pack [ComfyUI-MiniMax-H3-PDD-Acc](https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc), for **480p generation only**
- Stage persistence: nested-latent + conditioning save/load (see **Persistence between stages** below — method + links only; **no node code in this repo**)
- Latent upscale (nested H3 latent) → cached-conditioning refine → RTX Video Super Resolution (VSR)
- Orchestration: small Python submitters that POST API-format graphs to ComfyUI's local HTTP API (`/prompt`) and poll `/history` for completion and timings

## Current preferred mainline (quality / time)

1. **Gen 480p** — PDD Acc **nfe=8**, refs **pre-split**; **downscale only if short-edge >≈1024**, then `ref_image_size=max` (likeness). Pure `match` is faster if you can accept softer identity/hair
2. **Bake** 768-cond once (TE) — reusable across refine retries (~35–50s)
3. **Latent uponly** → **1344×768** (768p is Ref2VA's native training resolution and both axes are ÷32; avoid exact 1280×720 — height 720 is not ÷32; use **1280×704** if you need ~720p)
4. **Refine** — **R2V turbo 4-step (R2V4)** on cached cond, denoise **~0.75**, **kitchen dense** (no BlockSparse/SLA), euler, CFG 1.0
5. **VSR** — `HIGHBITRATE_ULTRA` **2×** (2688×1536). 3×/4K often looked too harsh/jaggy on clean anime lines
6. **VSR Denoise / Deblur** — left **off** for clean line art (A/B was inconclusive / not useful)

Optional slower quality bump: **LX8** refine instead of R2V4 (~+100s on an 8s 768 clip in our runs).

## Timing snapshots (8.0s = 192 frames @ 24fps)

Same machine, Ref2VA, single 5090. “match” = scale refs toward gen canvas; “max” = allow large short-edge (up to ~2048) — tokens every step.

| Setup | TOTAL wall | Notes |
|---|---:|---|
| ~12s @ 704, free_gpu on | ~649s | A_gen ~231s |
| ~12s @ 704, free_gpu off | ~635s | Skipping free saved only ~14s |
| 8s @ 704, all-max refs | ~446s | A ~181s |
| 8s @ 768, 480=match / bake=max, R2V4, VSR 3× | ~370s | A ~81s (fastest gen, but softer identity/hair) |
| 8s @ 768, LX8, match/match, VSR 2× | ~471s | LX8 refine alone ~290s |
| 8s @ 768, refs split + downscaled only above ~1024 short-edge + `max`, R2V4, VSR 2× | **~353s** | **A ~131s — current mainline recipe** (between match & full-size max) |

Stage example (8s / 768 / ref short-edge capped + `max` / R2V4 kitchen / ULTRA 2×):

`A_gen 130.6s · B_bake 35.7s · C_uponly 10.1s · D_r2v4 140.2s · E_vsr 36.8s → TOTAL 353.4s`

Locked kitchen mainline (same recipe, later clip, wall ≈ **5.5 min / ~330s**):

`A ~110s · B ~26s · C ~15s · D_r2v4_kitchen ~135s · E_vsr ~44s`

Denoise is not shown in the rows above; the refine-only walls measured per denoise are in the "Refine distill comparison" table (d0.5 ≈ 140s, d0.75 ≈ 150–160s). Expect the D stage to land in that band depending on the denoise you pick.

### What actually moved the needle

- **`ref_image_size=match` on 480 gen** — largest single speedup (gen ~180s → ~80s class on 8s)
- **Split views + short-edge cap ≈1024 (downscale only), then `max`** — preferred likeness/speed compromise (vs raw giant `max` or pure `match`)
- **Bake once**, reuse for refine A/Bs
- **`free_gpu` every stage** — only ~10–15s; not worth obsessing on 5090 if VRAM is fine
- Width micro-cuts (e.g. 1216 vs 1344) — little gain while DiT/refine dominates

### Reference images: `max`, but preprocess first

We prefer **`ref_image_size=max`** for likeness (hair silhouette / identity), **not** dumping giant raw sheets into Comfy.

Practical recipe:

1. **Split** multi-view character sheets into separate files (one image per view)
2. If a view’s **short edge is larger than ~1024**, **downscale** to short-edge ≈1024. If it is already smaller, **do not upscale** — stretching invents detail and is not the goal
3. Then set **`ref_image_size=max`** at gen (and at TE bake when baking 768-cond)

Why: full-size `max` on huge refs is the slowest (tokens every step). Pure `match` is fastest but softer on hair/identity. **Split + cap short-edge~1024 + `max`** is the middle ground we actually use.

## Refine distill comparison (same 8s / 1344×768 uponly + baked 768-cond)

| Refine | Wall (refine only) | Quality note |
|---|---:|---|
| **R2V4** d0.5 | **~140s** | Prior baseline |
| **R2V4** d0.75 | **~150–160s** | Preferred look (less soft than 0.5; less rewrite than 1.0) |
| **R2V4 + BlockSparseAttention** top-k 10% (same d0.75) | **~100–120s** | Faster, but **rejected** for mainline (look regressions on later clips) |
| **LX8** d0.5–1.0 | **~250–270s** | Heavier; keep as optional quality bump |
| **PDD Acc nfe=4** (PDD Scheduler) | **~60–130s** | Grainy / weak — **rejected for 768 refine** |
| **PDD Acc nfe=8** d0.25 | **~490s** | The upscale-refine example shipped with the community PDD node pack; slow and not better than R2V4 here |

### PDD Acc refine lessons

- Use **`MiniMaxH3PDDAccApply` + `MiniMaxH3PDDAccScheduler`** (trained sigma grid). **Do not** drive PDD with a plain `BasicScheduler` — off-grid evaluation is known to render as heavy noise.
- Same Acc-8Step file supports **`nfe=4|6|8`** in the community loader (4 regroups the trained blocks; check the pack's README for the exact sigma grid).
- The latent-upscale refine example in the community PDD node pack's workflows: **nfe=8 + denoise 0.25** (= last 2 trained blocks). On nfe=4, denoise 0.5 is “2 coarse steps” but each step is thicker — our run looked **much noisier** than R2V4.
- **Do not stack** turbo/distill LoRAs (R2V4 / LX8) on top of PDD Acc.
- Pair **Ref2VA Acc** with a **ref2va** UNET (trunk fingerprint guard).

## VSR notes (anime / clean lines)

- Prefer **HIGHBITRATE_ULTRA 2×** over 3×/4K for this look
- Denoise / Deblur A/B before ULTRA: **not adopted** (hard to judge / not needed)

## Resolution cheat sheet

| Target | Use |
|---|---|
| ~480p gen | 864×480 |
| ~720p latent | **1280×704** (not 1280×720) |
| Native 768p | **1344×768** |
| Deliverable | ULTRA 2× → 2688×1536 |

Two grid rules to plan around:

- **Width and height must both be multiples of 32** (hence 704, not 720) — off-grid sizes fail in the patchify step
- **Frame count lives on the `5 + 17k` grid** at 24 fps: 124, 141, 158, 175, **192 (= 8.0s)**, 209, …, 294, …, 362 (≈15s). Current ComfyUI core does not reject other values; it **snaps the requested length up** to the next grid point (see `align_frame_count` in `comfy_extras/nodes_minimax_h3.py`), so ask for a grid value explicitly or your clip comes out longer than planned

## Recipe constants that matter

- Sampler: **euler**
- Guidance: **CFG 1.0** (distills)
- Gen SigmaShift: **12.0 / 3.0** (turbo refine LoRAs use their own shift)
- Prefer Comfy HTTP polling that does not stall the UI for tens of seconds mid-run

## Persistence between stages (method + links; no node code here)

H3 stages do **not** hand off cleanly if you only keep the final mp4. Persist **nested latents** and **baked conditioning** between jobs so refine / A/B can reload without re-running TE or 480 gen.

This repo **does not ship custom nodes** (license / maintenance risk). Implement locally or pull third-party packs yourself.

### Why stock `SaveLatent` is not enough

H3 latents are a **`NestedTensor`** (video + audio members). See Comfy’s H3 extras and nested-tensor helpers:

- [comfy_extras/nodes_minimax_h3.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py)
- [comfy/nested_tensor.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy/nested_tensor.py)

Stock latent save assumes a plain tensor and will choke on nested AV latents. The sampler already **packs / unpacks** via `comfy.utils.pack_latents` / `unpack_latents` — mirror that for disk I/O.

### Nested latent save/load (recipe)

1. After gen / uponly / refine, take `LATENT["samples"]`
2. If nested: `unbind()` → `pack_latents(members)` → write one packed tensor plus per-member shapes (int64)
3. Store as **safetensors** (or equivalent) under a dedicated output subfolder (we use `output/h3_latents/`)
4. On load: read packed tensor + shapes → `unpack_latents` → rebuild `NestedTensor` → wrap as `{"samples": …}`

Treat the nested latent as the **source of truth** between stages (mp4 decode → re-encode is not equivalent). Tag filenames by stage/resolution so orchestrators can find them.

When / if Comfy lands first-class nested-latent Save/Load in core, prefer that and delete any local helper.

### Baked conditioning (768-cond) recipe

1. Run TE once at **768** encode size (same prompt, refs, length as refine) — no DiT sample
2. Persist the Comfy **CONDITIONING** tree (tensors + extras) to disk (e.g. `torch.save` of a CPU-moved tree under `output/h3_conds/`)
3. On refine: load that file and skip TE

Cond must match **prompt + refs + encode resolution + frame length**. Change any → rebake. Bake once, reuse across denoise / distill A/Bs on the same uponly latent.

Note: loading the conditioning file back requires `torch.load(..., weights_only=False)` (pickle). Only ever load files you baked yourself — never a `.pt` from someone else.

### VRAM / TE unload

Optional node **`H3FreeTextEncoder`** (free TE after building cond, before DiT) lives in third-party pack:

- [ComfyUI-H3-Multishot](https://github.com/jlucasmcrell/ComfyUI-H3-Multishot) (`h3_advanced.py`)

Check that repo’s license before installing. From an orchestrator, a plain `POST /free` with `{"unload_models": true, "free_memory": true}` between stages covers most of the same need without that node.

### Suggested stage I/O

1. **Gen 480** → save nested latent (+ optional mp4 preview)
2. **Bake 768-cond** → save conditioning (TE only)
3. **Upscale latent** → load 480 nested → uponly → save 1344×768 nested
4. **Refine** → load nested + conditioning → R2V4 → save nested + mp4
5. **VSR** → external upscaler on the refine mp4

Without steps 1–4 as files, every retry re-pays TE and/or 480 gen.

## Sparse Attention — tried, **not** mainline

We A/B’d Comfy **`BlockSparseAttention`** (top-k ~10%) on R2V4 refine and on full gen+refine. It was **faster** (~25% on refine) but **hurt look** on at least one character (noise / mess). Locked mainline stays **kitchen dense**; leave core sparse hooks / the node **out** unless you are deliberately experimenting.

## Still open

- Publishing sanitized orchestration scripts (parameterized paths; no personal assets)
- Whether a future upstream sparse path becomes look-safe enough to revisit

## License reminder

Base model and the Acc LoRAs follow their upstream licenses. Note that alibaba-pai's `MiniMax-H3-Acc-LoRAs` repo switched its license field from Apache-2.0 to the **MiniMax-H3 Community License Agreement** on 2026-08-27 — check the [current model card](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs) and its [commit history](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/commits/main), not older mirrors. Third-party packs you install yourself (e.g. Multishot, PDD Acc loader) keep **their** licenses — this notes repo ships docs only. Verify commercial use for your jurisdiction before shipping product.

---

*Numbers from one RTX 5090 workstation, 2026-09.*
