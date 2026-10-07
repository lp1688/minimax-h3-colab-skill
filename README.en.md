# MiniMax H3 Colab Skill

This repository is a complete, standalone Codex skill for creating short MiniMax H3 Ref2VA videos from local reference images through Google Colab. Clone it, install the skill, authenticate the Colab CLI, and invoke the included runner or shell launcher. The repository contains:

- `SKILL.md`: the instructions Codex loads when this skill is selected;
- `scripts/runner.py`: the session, usage, upload, batch, download, and cleanup runner;
- `assets/MiniMax_H3_Turbo_Colab.ipynb`: the notebook executed on the remote Colab runtime;
- `run_colab_inference.sh`: a convenient single-video launcher;
- `install.sh`: a portable, non-destructive skill installer;
- `tests/test_runner.py`: offline tests using a fake Colab CLI.

The local machine only prepares and uploads inputs. Model inference runs on the Colab GPU and the finished MP4 is downloaded back to the path you choose.

## Requirements

- A working Codex installation. The skill is installed into `$CODEX_HOME/skills` when `CODEX_HOME` is set, otherwise into `~/.codex/skills`.
- Python 3.11 or newer for the bundled runner. The runner uses only the Python standard library.
- [`uv`](https://docs.astral.sh/uv/) for installing the Colab CLI, or another supported way to put `colab` on `PATH`.
- [`google-colab-cli`](https://pypi.org/project/google-colab-cli/). The runner was validated with Colab CLI 0.7.4 and uses the documented `version`, `usage`, `new`, `upload`, `exec`, `download`, and `stop` commands. The current CLI release requires Python 3.12 or newer; `uv` can install that interpreter separately from the runner's Python 3.11+ requirement.
- A Google account with access to Colab compute units and a GPU shape that can be allocated. An A100 or equivalent high-memory runtime may require the appropriate Colab plan and available balance.

The optional `ffprobe` program is used to verify that a downloaded MP4 contains both video and audio streams. If `ffprobe` is not installed, the runner still checks that the file exists and is non-empty.

## Clone and install the skill

```bash
git clone <repository-url> minimax-h3-colab-skill
cd minimax-h3-colab-skill
./install.sh
```

The installer copies the required skill files to:

```text
$CODEX_HOME/skills/minimax-h3-colab
```

When `CODEX_HOME` is unset, the destination is `~/.codex/skills/minimax-h3-colab`.

Use an explicit skills directory when testing or when your Codex configuration is elsewhere:

```bash
./install.sh --dest /absolute/path/to/codex/skills
```

The default operation is idempotent and non-destructive. If the destination already exists, the installer leaves it unchanged and exits successfully. To replace it deliberately, use `--force`; the old directory is first moved to a timestamped `.backup.*` path so it can be recovered:

```bash
./install.sh --force
```

The installer uses a temporary directory and an atomic rename for a new installation. It never copies the repository's Git metadata, tests, README files, or generated output into the installed skill; only `SKILL.md`, `scripts/`, and `assets/` are installed.

After installation, start a new Codex turn or reload the skill list if your Codex client caches available skills. The skill name is `minimax-h3-colab`.

## Install and authenticate Colab CLI

Install the CLI as a user tool:

```bash
uv python install 3.12
uv tool install --python 3.12 google-colab-cli
```

Confirm that it is available:

```bash
colab version
```

The runner defaults to OAuth2. Trigger the first authorization and inspect the account balance with:

```bash
colab --auth=oauth2 usage
```

Follow the URL and code instructions printed by the CLI. The token is stored by the CLI in its normal local configuration; do not put credentials, tokens, or browser state in this repository, a prompt, or a job manifest. If your environment already uses Google Application Default Credentials, choose the alternative provider explicitly:

```bash
COLAB_AUTH=adc colab --auth=adc usage
```

The runner passes `--auth="$COLAB_AUTH"` to every Colab command. Its default is `oauth2`.

## Single-video inference

Create a UTF-8 text file such as `prompt.txt`, then run:

```bash
./run_colab_inference.sh \
  --image /absolute/path/reference_1.png \
  --image /absolute/path/reference_2.jpg \
  --prompt /absolute/path/prompt.txt \
  --output /absolute/path/intro.mp4
```

The first image is exposed to the notebook as `<Picture 1>`, the second as `<Picture 2>`, and so on. Pass 1–9 non-empty image files. The prompt file must be non-empty UTF-8 text and is uploaded unchanged. If `--output` is omitted, the MP4 is written next to the first reference image with a `_minimax_h3.mp4` suffix.

The default clip length is 12 seconds. Set `H3_DURATION_SECONDS` to a value from 4 through 15:

```bash
H3_DURATION_SECONDS=8 ./run_colab_inference.sh \
  --image /absolute/path/reference.png \
  --prompt /absolute/path/prompt.txt
```

The shell launcher requests an A100 high-memory runtime by default. These environment variables change the request:

| Variable | Default | Meaning |
| --- | --- | --- |
| `COLAB_AUTH` | `oauth2` | Colab CLI auth strategy: `oauth2` or `adc` |
| `COLAB_GPU` | `A100` | GPU name passed to `colab new` |
| `COLAB_HIGH_MEM` | `1` | Set to `0` to omit `--high-mem` |
| `COLAB_EXEC_TIMEOUT` | `3600` | Per-notebook execution timeout in seconds |
| `COLAB_SESSION_NAME` | generated | Reusable session name for the runner |
| `H3_DURATION_SECONDS` | `12` | Clip duration, from 4 to 15 seconds |

## Batch inference

For several clips, keep all jobs in one manifest so they reuse one Colab session and the loaded model. Example `jobs.json`:

```json
{
  "jobs": [
    {
      "id": "intro",
      "title": "Presenter introduction",
      "reference_images": [
        "/absolute/path/girl-front.png",
        "/absolute/path/girl-side.png"
      ],
      "prompt_file": "/absolute/path/intro-prompt.txt",
      "duration_seconds": 8,
      "output_name": "intro"
    },
    {
      "id": "demo",
      "title": "CLI demo",
      "reference_images": ["/absolute/path/girl-front.png"],
      "prompt": "A concise Ref2VA prompt referring to <Picture 1>.",
      "duration_seconds": 12,
      "output_name": "demo"
    }
  ]
}
```

Run the batch from the cloned repository:

```bash
python3 scripts/runner.py batch \
  --manifest /absolute/path/jobs.json \
  --gpu A100 \
  --timeout 10800 \
  --output-dir /absolute/path/outputs \
  --progress /absolute/path/outputs/progress.json
```

The runner validates all local inputs before starting. It uploads each job's references and prompt, executes the bundled notebook, downloads and verifies the MP4, and continues with the next job. A later failure does not delete completed outputs. A session created by the batch is stopped during cleanup, including after an error or timeout. If you pass `--session NAME` to reuse an existing session, add `--stop-on-complete` when that session should be stopped after the queue.

The installed skill can also be invoked directly without the cloned repository:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/minimax-h3-colab/scripts/runner.py" \
  batch --manifest /absolute/path/jobs.json --output-dir /absolute/path/outputs
```

## Prompt and image rules

- Each job needs 1–9 non-empty local reference images.
- Image order is stable and is the only mapping used for `<Picture N>` tags.
- A prompt file is read as UTF-8 and sent as the complete prompt; the runner does not translate, summarize, or rewrite it.
- A prompt cannot refer to a picture number greater than the number of images uploaded for that job.
- Each video duration must be between 4 and 15 seconds.
- For guided prompts in another application, use the runner's `compose_ref2va_prompt` helper to produce `subject_definitions`, `summary`, `retention_analysis`, ordered shot blocks, `overall_soundscape`, and `non_diegetic_music`.
- The reference image guide is available in the [MiniMax H3 Ref2VA prompt guide](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_ref_en.md).

Reference images describe visual identity and appearance. They do not automatically create shot timestamps; describe timing and camera changes in the prompt.

## Direct notebook use

`assets/MiniMax_H3_Turbo_Colab.ipynb` is the exact notebook the runner uploads and executes. It can also be opened manually in Colab for inspection or debugging. The runner's environment variables select reference mode, remote image paths, prompt path, duration, seed, output path, and execution timeout. Keep the notebook copy in this repository with the runner so the two stay compatible.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| `colab` is missing | Run `uv tool install google-colab-cli`, then ensure the uv tool bin directory is on `PATH`. |
| OAuth or usage fails | Run `colab --auth=oauth2 usage` interactively and complete the printed Google authorization flow. |
| GPU allocation fails | Check Colab plan, compute-unit balance, requested GPU, and high-memory availability. Try `--no-high-mem` or another supported GPU. |
| A batch times out | Do not blindly retry while the remote kernel may still be running. The runner attempts to stop a session it owns and records completed jobs in progress state. |
| Prompt picture validation fails | Match `<Picture N>` to the 1-based order of the 1–9 `reference_images` entries. |
| Output exists but is rejected | Install `ffprobe` and inspect the downloaded file's video and audio streams. |

## Offline validation

No GPU or Colab session is needed to run the repository tests:

```bash
python3 -m unittest discover -s tests -v
python3 scripts/runner.py --help
./run_colab_inference.sh --help
```

The tests replace `colab` and `ffprobe` with local fakes, so they do not spend compute units or access credentials.

## Scope and safety

This repository does not contain Google credentials, tokens, model weights, or generated videos. Colab sessions consume the account's compute units. Do not place secrets in prompts, manifests, logs, or uploaded files. Review the requested GPU and timeout before starting a real batch.

## Fork additions (lp1688)

This fork extends the upstream skill with two areas of changes, both validated in production runs on a Windows 11 + Git Bash machine driving Colab through `google-colab-cli` 0.7.4. The skill also works with Kimi Code (installed with `./install.sh --dest ~/.kimi-code/skills`); nothing in the skill is Codex-specific.

### Windows support

The upstream workflow targets macOS/Linux. These are the verified adjustments for Windows:

- **`python3` command**: Windows Python ships only as `python`. Put a small shim at `~/.local/bin/python3` (`#!/usr/bin/env bash` + `exec python "$@"`) so `install.sh` and the runner work unchanged.
- **Two patches to `google-colab-cli` 0.7.4** (under `%APPDATA%\uv\tools\google-colab-cli\Lib\site-packages\colab_cli\`):
  1. `console.py`: `import termios` / `import tty` are Unix-only. Wrap them in `try/except ImportError` and set both names to `None`; only the interactive console/ssh feature uses them.
  2. `commands/automation.py`: the Drive auth flow waits on `open("/dev/tty")`, which does not exist on Windows. Fall back to `sys.stdin.readline()` on `OSError` (an EOF continues immediately).
  Re-running `uv tool install --force` or `uv tool upgrade` wipes these patches; reapply them afterwards.
- **MSYS path conversion**: Git Bash rewrites arguments that look like Unix paths, so `colab drivemount ... /content/drive` becomes `C:/Program Files/Git/content/drive`. Export `MSYS_NO_PATHCONV=1` before calling `colab` directly with remote paths. Calls made inside `runner.py` are unaffected because Python's `subprocess` performs no conversion.
- **Console encoding**: the Windows console default code page (cp950/cp936) crashes Python when the runner prints notebook output containing other characters. Run the runner with `PYTHONIOENCODING=utf-8` (or `PYTHONUTF8=1`). Without this, a batch can complete successfully yet exit with a misleading encoding error at the final print.
- **`os.killpg` fix (included in this fork's `runner.py`)**: `os.killpg` does not exist on Windows, which crashed the timeout-kill path and masked the original error. The runner now falls back to `child.terminate()` / `child.kill()`.
- **Offline tests**: 4 of the 8 repository tests fail on Windows because the fake `colab`/`ffprobe` fixtures are extension-less shell scripts that Windows cannot execute (`WinError 193`) and `shutil.which("colab")` cannot find. This is a test-harness limitation; the runner itself was verified against the real CLI.

### Google Drive persistent model cache

A new session normally re-downloads the full model set (~38 GiB) from Hugging Face. This fork adds an optional persistent cache on Google Drive:

```bash
python3 scripts/runner.py batch \
  --manifest /absolute/path/jobs.json \
  --drive-cache /content/drive/MyDrive/minimax-h3-models \
  --output-dir /absolute/path/outputs
```

`single` and the `H3_DRIVE_CACHE` environment variable work the same way. Behavior:

1. The runner mounts Google Drive on the session with `colab drivemount`. Because `colab exec` exits 0 even when remote code raises, the mount is verified by executing a probe that prints a marker (`H3_DRIVE_MOUNT_OK`); the runner retries the mount up to 3 times and aborts the batch if Drive never mounts.
2. The notebook asserts `os.path.ismount('/content/drive')` before using the cache, so a failed mount can never silently fill the VM's ephemeral local disk with 38 GiB of weights.
3. For each model file, a cache hit is copied from Drive to the VM's local disk (`shutil.copy2`); a miss is downloaded from Hugging Face and pushed back to the cache via a temporary `.partial` file and an atomic rename. The cache layout mirrors `ComfyUI/models/` (`diffusion_models/`, `text_encoders/`, `vae/`, `loras/`).

**Per-VM interactive grant (important).** Drive authorization is tied to the individual VM endpoint, not to the Google account. Every new Colab session needs one browser approval:

```bash
colab drivemount --session SESSION /content/drive   # prints an authorization URL
# open the URL in a browser and approve (about 5 seconds)
colab drivemount --session SESSION /content/drive   # second run propagates and mounts
```

Granting Drive access in the Colab web UI does not remove this requirement; it was verified that a fresh VM still asks for approval. Because the CLI waits for Enter on stdin after printing the URL, non-interactive runners must treat the first `drivemount` call as a probe that surfaces the URL, then re-run it after the user approves.

**Measured performance (A100 high-mem, ~38 GiB model set):**

| Model source | Total batch time (one 4 s clip) |
| --- | --- |
| Hugging Face download on a fresh session | ~7.5–9.5 minutes |
| Drive cache hit (copy Drive → VM) | ~20.5 minutes |

Drive FUSE reads are much slower than the Hugging Face CDN on Colab, so the cache is **not** a time saver. Recommended usage:

1. **Default:** omit `--drive-cache`; re-downloading from Hugging Face is faster and fully reliable (~0.6 compute units of overhead per new session).
2. **Same working period:** reuse a named session (`--session NAME` across batches). The model loads once and stays in memory/disk, which is the real time saver. Stop the session when finished; an idle A100 bills about 6.77 compute units per hour.
3. **Use `--drive-cache` as a fallback** when Hugging Face is rate-limiting or unreachable, accepting the slower load.

The full model set needs roughly 38 GiB of Drive space; check the Drive quota before the first warmup run.

### Switching MiniMax H3 model variants

The notebook picks its weights automatically, but these environment variables override the selection (the runner forwards them when set):

| Variable | Values | Effect |
| --- | --- | --- |
| `H3_REF2VA_VARIANT` | `int8_convrot` (default on A100), `fp8_scaled`, `bf16` | Ref2VA diffusion weights. `fp8_scaled` requires GPU capability ≥ 8.9 (A100 is 8.0 and cannot use it); `bf16` needs ~80 GiB free disk and more VRAM |
| `H3_DIFFUSION_VARIANT` | `auto`, `fp8_scaled`, `int8_convrot` | FL2VA (`first_frame` mode) diffusion weights |
| `H3_TEXT_ENCODER_REPO` / `H3_TEXT_ENCODER_REMOTE` | Hugging Face repo / file path | Swap the Qwen3-VL text encoder (e.g. an abliterated "heretic" build). The file lands in `models/text_encoders/` and the workflow's CLIPLoader picks it up automatically |
| `H3_LORA_REPO` / `H3_LORA_REMOTE` | Hugging Face repo / file path | Swap the Turbo LoRA |
| `H3_LORA_STRENGTH` | float, default 1.0 | LoRA strength (sane range ~0.8–1.2) |
| `H3_STEPS` | integer (Ref2VA default 4, FL2VA default 8) | Sampler steps |

Example — highest-quality Ref2VA run on an A100:

```bash
H3_REF2VA_VARIANT=bf16 H3_STEPS=6 python3 scripts/runner.py batch \
  --manifest /absolute/path/jobs.json --output-dir /absolute/path/outputs
```

Example — swap in an abliterated ("heretic") text encoder:

```bash
H3_TEXT_ENCODER_REPO=ethanfel/Qwen3-VL-32B-Ultra-Heretic-H3-ComfyUI-INT8-ConvRot \
H3_TEXT_ENCODER_REMOTE=qwen3vl_32b_h3_ultra_uncensored_heretic_int8_convrot.safetensors \
python3 scripts/runner.py batch --manifest /absolute/path/jobs.json --output-dir /absolute/path/outputs
```

The text encoder is the component that interprets (and can refuse) prompts; community "heretic"/abliterated builds remove refusal behavior. Only the encoder changes — diffusion weights, VAEs, and the LoRA stay as configured.

**Two builds are verified working on A100 (sm_80):** the INT8-ConvRot build above (~26 GB) and the smaller NVFP4 build [Momoking/Qwen3-VL-32B-Heretic-MiniMax-H3-NVFP4](https://huggingface.co/Momoking/Qwen3-VL-32B-Heretic-MiniMax-H3-NVFP4) (~15.7 GB, remote filename `qwen3vl_32b_heretic_minimax_h3_nvfp4.safetensors`). The NVFP4 build's README warns that NVFP4 needs Blackwell-generation hardware, but ComfyUI loads its `TensorCoreNVFP4Layout` through a software path on A100 and produced prompt-accurate output in our tests; prefer it for the 40% smaller download. You are responsible for complying with the Colab and Hugging Face acceptable-use policies when running modified models.

Variants use distinct filenames, so the Drive cache stores them side by side; switching variants does not invalidate previously cached files. Swapping to a completely different model family (Wan, LTX, HunyuanVideo, …) is out of scope: the notebook is built around the ComfyUI H3 nodes, its VAEs, and the Ref2VA prompt format.
