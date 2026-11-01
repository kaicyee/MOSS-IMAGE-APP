# Running MOSS Image

Four independent ways to generate images, and where the files end up. None of them depend on
each other — the MCP integration in particular is a convenience layer, not a requirement.

## 1. Web UI

Double-click `Open MOSS Image.command` on the Desktop, or run `./start_ui.sh`. The page opens at
`http://127.0.0.1:7877` (`MOSS_UI_PORT`).

This is the full app: gallery, queue, live progress, settings, plus the Edit (img2img and
inpainting) and Layout (bounding-box composition) tabs. The backend on port 18765 is started on
demand the first time you generate, or reused if it is already running.

To stop it, close the browser tab and press `Ctrl-C` in the terminal window that opened.

## 2. Command line

```bash
cd ~/Desktop/moss-image-app
./generate.sh "a red bicycle in fog"
```

Full signature is `./generate.sh "prompt" [width] [height] [seed] [quality]`:

```bash
./generate.sh "a red bicycle in fog" 1024 1024 -1 standard
```

| `quality`  | Steps | Notes                            |
|------------|-------|----------------------------------|
| `draft`    | 3     | 3-step turbo LoRA, fastest       |
| `standard` | 4     | 4-step turbo LoRA, the default   |
| `high`     | 50    | official settings, CFG 3, slow   |

A seed of `-1` is random; any fixed integer reproduces the same image.

For more control, call the client directly — it takes `--init`, `--mask`, `--strength`,
`--layout`, `--cutout`, `--upscale` and `-o`:

```bash
.venv/bin/python moss_client.py "a prompt" --width 1024 --height 1024 -o ~/Pictures/out.png
```

## 3. MCP clients

`claude mcp add moss-image -s user -- "$PWD/.venv/bin/python" "$PWD/moss_mcp.py"`

Exposes six tools: `generate_image`, `generate_layout_image`, `generate_cutout_object`,
`edit_image`, `transform_image` and `image_server_status`. See [mcp-clients.md](mcp-clients.md)
for client-specific configuration.

Restart the client after adding the server or the tools will not appear.

## 4. HTTP API

```bash
./start_server.sh
```

Three APIs on `http://127.0.0.1:18765` — the native async endpoint, an A1111-compatible one, and
an OpenAI-compatible `/v1/images/generations`. It has **no authentication**; keep `MOSS_HOST` at
`127.0.0.1` unless you put your own proxy in front of it. Details in [http-api.md](http-api.md).

## Keeping the backend warm

A cold `./generate.sh` pays to load the 4-bit DiT, the Qwen3.5-9B text encoder and the VAE before
it samples anything — on a 16 GB M1 Pro that is several minutes for the first 1024x1024 image.

Running `./start_server.sh` keeps all three resident so each request only pays for sampling.
`generate.sh` probes `/sdcpp/v1/capabilities` on startup and routes through the server when it
finds one, so no second copy goes into GPU memory. The UI and the MCP server manage this
automatically. See [performance.md](performance.md).

## Where images are written

Default is `outputs/` inside the repo, with timestamped names like `moss_20261101_093653.png`.

Three hidden subdirectories live alongside them:

| Path               | Contents                                                    |
|--------------------|-------------------------------------------------------------|
| `outputs/.meta/`   | `<name>.png.json` per image — prompt, seed and settings used |
| `outputs/.uploads/`| images uploaded for editing or inpainting                    |
| `outputs/.tmp/`    | scratch space for the MCP server                             |

There is no configuration file for the output path. It is set in source, and each entry point
resolves it differently:

| Entry point       | Defined at                                            | Override                 |
|-------------------|-------------------------------------------------------|--------------------------|
| Web UI            | `moss_ui.py:29` — `OUTPUTS = os.path.join(ROOT, "outputs")` | none (hardcoded)   |
| MCP server        | `moss_mcp.py:34` — `MOSS_OUTPUT_DIR`, default `outputs/`    | `MOSS_OUTPUT_DIR`  |
| `generate.sh`     | `generate.sh:19`                                       | none (hardcoded)        |
| `moss_client.py`  | `moss_client.py:657`                                   | `-o` / `--out`          |

To send MCP output somewhere else, re-register the server with the variable set:

```bash
claude mcp remove moss-image -s user
MOSS_OUTPUT_DIR=~/Pictures/moss claude mcp add moss-image -s user -- \
  ~/Desktop/moss-image-app/.venv/bin/python ~/Desktop/moss-image-app/moss_mcp.py
```

Repointing the UI means editing `moss_ui.py:29`, which is worth avoiding: the gallery's delete and
upscale handlers compare against `OUTPUTS` and refuse to touch files outside it, so moving the
constant changes those guards too. To browse other folders without that risk, list them in
`MOSS_EXTRA_GALLERY_DIRS` (`moss_ui.py:43`) — they render read-only.

## Backups

`outputs/` sits inside the git clone and is ignored, not tracked. A `git clean` or a fresh clone
takes the images with it. Copy anything worth keeping out of the repo.
