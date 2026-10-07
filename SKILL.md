---
name: minimax-h3-colab
description: Create short MiniMax H3 reference-to-video clips from one or more local images through Google Colab CLI. Use when a user asks for image-guided H3 video generation or a sequential batch of clips.
---

# MiniMax H3 on Colab

Use the bundled notebook and `scripts/runner.py` to generate Ref2VA video from local reference images. The Colab CLI must be installed and authenticated with an available compute-unit balance and GPU allocation.

## Workflow

1. Inspect the requested images and any supplied prompt file. Keep image order stable: image 1 maps to `<Picture 1>`, image 2 to `<Picture 2>`, through image 9.
2. If the user supplied a prompt file, pass it through unchanged. Otherwise write a complete Ref2VA prompt with `subject_definitions`, `summary`, `retention_analysis`, `detailed_description` shot blocks, `overall_soundscape`, and `non_diegetic_music`; include dialogue in its shot with `<d>[Chinese] ...</d>` when requested. Use the bundled runner's `compose_ref2va_prompt` helper for structured shot input.
3. Check Colab access and balance with `python3 scripts/runner.py usage --json`. Sufficient compute units do not guarantee that Colab will allocate the requested GPU or high-memory runtime. Report allocation failures plainly.
4. Put the jobs in a UTF-8 JSON manifest and run one batch. The runner uploads each job's images and prompt, executes the bundled notebook sequentially on the same live session, downloads each MP4, and stops the session when it created it. If reusing a session, pass `--stop-on-complete` when the user wants it stopped after the batch.
5. Confirm each output exists and report its local path. Preserve completed clips if a later job fails.

Example manifest:

```json
{
  "jobs": [
    {
      "title": "intro",
      "reference_images": ["/absolute/path/girl.jpg"],
      "prompt_file": "/absolute/path/prompt.txt",
      "duration_seconds": 12,
      "output_name": "intro"
    }
  ]
}
```

Run from this skill directory:

```bash
python3 scripts/runner.py batch --manifest /absolute/path/jobs.json --gpu A100 --timeout 10800 --output-dir /absolute/path/outputs
```

Use `--no-high-mem` when high-memory allocation is unavailable or not desired. For a named existing session, add `--session SESSION --stop-on-complete` if the requested workflow should end that session.

## Persistent model cache on Google Drive

Pass `--drive-cache /content/drive/MyDrive/minimax-h3-models` (or set `H3_DRIVE_CACHE`) to `batch` or `single`. The runner mounts Google Drive on the session with `colab drivemount` before executing; the notebook then copies cached model weights from Drive to the VM's local disk and pushes fresh Hugging Face downloads back to Drive, so new sessions skip the multi-GB re-download. The first Drive authorization is per Google account: if the CLI prints an authorization URL, open it in a browser and grant access once; later mounts are non-interactive. The Drive account needs enough free space for the full model set (roughly 30 GB). Without `--drive-cache` nothing is mounted and behavior is unchanged.

## Model variants (H3 family)

Optional environment variables switch weights inside the MiniMax H3 family; the runner forwards them to the notebook when set:

| Variable | Values | Effect |
| --- | --- | --- |
| `H3_REF2VA_VARIANT` | `int8_convrot` (default on A100), `fp8_scaled`, `bf16` | Ref2VA diffusion weights. `fp8_scaled` needs GPU capability ≥ 8.9 (not A100); `bf16` needs ~80 GiB free disk and more VRAM |
| `H3_DIFFUSION_VARIANT` | `auto`, `fp8_scaled`, `int8_convrot` | FL2VA (`first_frame` mode) diffusion weights |
| `H3_LORA_REPO` / `H3_LORA_REMOTE` | Hugging Face repo / file path | Swap the Turbo LoRA |
| `H3_LORA_STRENGTH` | float, default 1.0 | LoRA strength (sane range ~0.8–1.2) |
| `H3_STEPS` | integer (Ref2VA default 4, FL2VA default 8) | Sampler steps; more steps is slower |

Different variants have distinct filenames, so the Drive cache holds them side by side without conflicts.

## Operational limits

- Each job requires 1–9 non-empty reference images, a non-empty UTF-8 prompt, and a duration from 4–15 seconds.
- Keep all jobs in one batch to reuse the same Colab session and loaded ComfyUI/model state.
- Do not blindly retry a timed-out `colab exec`: the remote kernel may still be working. The runner stops the session during cleanup and records completed jobs before reporting the failure.
- Do not expose OAuth credentials or runtime tokens in prompts, manifests, browser state, or logs.
- The bundled `assets/MiniMax_H3_Turbo_Colab.ipynb` is the inference notebook used by the runner.
