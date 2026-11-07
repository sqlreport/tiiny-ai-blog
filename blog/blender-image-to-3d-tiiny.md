# Image to 3D in Blender with Tiiny: Setup, Fixes, and a macOS Port

By Tae Lee · November 7, 2026 · 14 min read

Blender’s Image to 3D add-on can send a reference image to a Tiiny device running Hunyuan3D-2.1 or TRELLIS.2-4B and bring the generated mesh back into your scene. The setup has two parts: configure Blender’s connection, and load the 3D model on the device. A downloaded model alone is not enough.

These notes cover my locally adapted **Image to 3D 1.1.5** extension with a Tiiny backend. The stock add-on is Windows-only and does not include these settings. This post does not distribute the adapted extension. Before following the Blender steps you need a Tiiny-enabled package, or you can build one yourself as described in [Adapting the Windows-only extension for macOS](#adapting-the-windows-only-extension-for-macos).

The post covers two fixes: the [HTTP 503 connection error](#fix-test-connection-returns-http-503), and a [model that comes out as a flat square](#fix-the-model-comes-out-as-a-flat-square).

## What you need

- Blender 4.2 or later, as declared by this extension’s manifest. The adapted package lists Windows x64 and macOS ARM64 support.
- The Tiiny-enabled Image to 3D extension ZIP.
- An activated, unlocked Tiiny device reachable from your computer over USB networking or your local network.
- TiinyOS service version 0.1.35 or later, according to the SDK documentation checked for this setup.
- The device’s IP address and API key, plus the Tiiny CLI for the commands below.
- A reference image with a clearly visible subject. A simple background makes a useful first test.

In this configuration, Hunyuan inference runs on the Tiiny device. On macOS, choose the Hunyuan engine in the add-on; the local CUDA engines are a separate workflow.

## 1. Connect to the device

Find the address under **Settings → Networks** in TiinyOS. Find the API key under **Settings → API Key**. Keep the key out of screenshots and published files.

Discover the device from a terminal:

```sh
tiiny scan
tiiny connect <device-ip>
```

Replace `<device-ip>` with the address reported for your device. Discovery may select the device automatically; an explicit connect is useful if several devices are available. If the device says it is locked, run `tiiny login` and finish signing in before loading a model.

## 2. Download and load Hunyuan3D

List downloaded models and loaded models separately:

```sh
tiiny ls
tiiny ls -l
```

For this device, the downloaded list contained Hunyuan3D-2.1, but the loaded list contained only Qwen. That difference explained the connection failure.

Use the exact model ID printed by your device. In my list it was `Tencent/Hunyuan3d-2.1`, with a lowercase `d` in the ID even though the display name is Hunyuan3D-2.1.

If it is missing from the downloaded list, download it first:

```sh
tiiny download Tencent/Hunyuan3d-2.1
```

Then load it and wait for completion:

```sh
tiiny load Tencent/Hunyuan3d-2.1
tiiny ls -l
```

Run the load command in an interactive terminal; this CLI’s loading display requires a TTY. Confirm Hunyuan3D appears in the loaded list before testing Blender. If loading fails, inspect that error first; the Blender connection check cannot start the model for you.

## 3. Install and configure the Blender extension

In Blender, open **Edit → Preferences → Get Extensions**, open the menu, and choose **Install from Disk**. Select your Tiiny-enabled extension ZIP. Enable **Image to 3D** under **Add-ons**, then expand its preferences. Menu placement can vary by Blender version.

In the Tiiny settings:

1. Enable **Use Tiiny AI for Hunyuan3D-2.1**.
2. Set **Base URL** to `http://<device-ip>/v1`, replacing the placeholder with your device’s address. Include `/v1`; do not append `/3d` here.
3. Choose **Key Storage** and provide the device’s **API Key**. The adapted extension supports Preferences, macOS Keychain, and an environment variable.
4. Leave **Auth Header** at **Authorization: Bearer** unless your deployment requires another header.
5. Leave **Remove Background on Tiiny** enabled for an ordinary photo. You can turn it off for an image already cut out. On my device this step currently fails and the job quietly continues with the original picture; see [the flat-square fix](#fix-the-model-comes-out-as-a-flat-square).
6. If Blender reports network access is disabled, enable **Preferences → System → Allow Online Access**.
7. Click **Test Connection** and wait for the result. Save your preferences if automatic saving is disabled.

For macOS Keychain storage, use the key-storage button shown after selecting that option. Preferences storage masks the key on screen but stores it as plain text in Blender’s preferences. The environment-variable option reads `IMG3D_TIINY_API_KEY` from Blender’s process environment; set it before launching Blender.

## 4. Generate your first mesh

Open the add-on’s **Image to 3D** viewport interface, select **Hunyuan**, and choose a reference image. Start with a modest quality setting and click **Generate 3D**. Watch the stage and progress display while the device works.

The API pipeline consists of background removal, shape generation, optional simplification, texturing, and export. The adapted backend downloads the resulting GLB and imports it into Blender. Inspect the geometry, materials, orientation, and scale before using the asset in a larger scene. Save the Blender project once you have a result you want to keep.

The connection and model-loading checks below were verified against the physical device. Complete generations were later verified with TRELLIS.2-4B; the timings are in [the flat-square fix](#fix-the-model-comes-out-as-a-flat-square).

## Fix: Test Connection returns HTTP 503

The error that prompted this guide was:

```text
No 3D model is running on Tiiny - load Hunyuan3D-2.1 on the device first (HTTP 503)
```

The device response was:

```json
{"error":{"message":"No suitable model is currently running.","type":"service_unavailable"}}
```

The API was reachable, but it could not route the request to a running 3D model. Having Qwen loaded did not satisfy the Image-to-3D endpoint. Hunyuan3D was already downloaded; loading it resolved the failure without reinstalling Blender or downloading the model again.

You can check the 3D service independently of Blender. Substitute your device’s IP in this command:

```sh
curl -i --max-time 15 http://<device-ip>/v1/3d/health
```

This health endpoint does not require an API key. Before loading Hunyuan3D, it returned **HTTP 503**. After loading completed, it returned **HTTP 200 OK** with:

```json
{"status":"ok","service":"image-to-3d_server","version":"2.0.0"}
```

That confirms the 3D service is available; Blender’s **Test Connection** still needs to verify the configured key and the other endpoints it uses. Click it again after the health check succeeds.

## Other connection failures

- **No device connected in the CLI:** run `tiiny scan`, then `tiiny connect <device-ip>`.
- **Timeout or connection refused:** check the current device IP, USB or network connection, and Blender’s network-access setting.
- **HTTP 401 or 403:** check the API key, selected key source, and authentication header.
- **HTTP 404:** check that the base URL ends in `/v1` and that the installed service supports Image-to-3D.
- **503 after a restart or model change:** inspect `tiiny ls -l` again and load Hunyuan3D if it is absent.

For the failure observed here, the repeatable fix was `tiiny load Tencent/Hunyuan3d-2.1`, followed by a successful health check and another Blender connection test.

## Fix: the model comes out as a flat square

The connection worked and generation finished, but the imported model was a flat square: the whole picture, background included, turned into a thin slab, with the subject painted on its front.

### Check what you actually got

Don't judge by the bounding box alone. My broken results measured about 2 × 2 × 1.4 units, which looks like a solid object. Hunyuan3D generates inside a cube from -1 to 1, so a result that touches the cube on two axes is suspicious. Split the mesh into loose parts (**Edit Mode → Mesh → Separate → By Loose Parts**) and look at the thickness. Mine was one or more flat sheets about 0.014 units thick, one voxel of the generation grid, or a scattering of small flat fragments. A healthy result is one solid piece shaped like the subject.

### What it was not

I worked through the likely causes in order. None of these fixed it:

- **Remove Background was off.** That is a real cause for photos with a solid background: Hunyuan3D treats every opaque pixel as part of the object. Turning it on did not help here, for the reason below.
- **The input image.** A properly cut-out PNG (70% of pixels fully transparent) gave the same result as the same cut-out flattened onto white.
- **Quality settings.** Draft and standard precision both came back flat.
- **Other models sharing the device.** Unloading Qwen so Hunyuan ran alone changed nothing.
- **A bad download.** `tiiny unload`, `tiiny rm`, `tiiny download`, and `tiiny load` for `Tencent/Hunyuan3d-2.1` gave a fresh store copy that failed identically.

### The cause: the 3D model on the device

To take Blender out of the picture, I sent each stage straight to the device's API and measured the meshes it returned. Two faults showed up on the device itself:

**The background stage fails** with:

```text
BriefNet ONNX model not found: /model/BriefNet
```

The extension treats a failed background stage as non-fatal. It shows "Background removal failed on Tiiny - using the picture as it is" in the progress line and sends the original picture to the shape stage, so the toggle in Blender has no effect.

**Hunyuan3D's shape stage returns flat one-voxel sheets** for every input. The 3D health check still says `ok` and `tiiny ls -l` shows the model loaded, so neither catches it.

The fault reproduced on a clean reinstall, so it lies in the model package or the device's 3D server (`image-to-3d_server` 2.0.0), not on the computer. I reported it to Tiiny support with the error above and the mesh measurements.

### Workaround: use TRELLIS.2-4B

TRELLIS.2-4B, a second Image-to-3D model in the Tiiny store, produced correct meshes from the same image on the same device and server:

```sh
tiiny unload Tencent/Hunyuan3d-2.1
tiiny download microsoft/TRELLIS.2-4B   # skip if `tiiny ls` already lists it
tiiny load microsoft/TRELLIS.2-4B
```

TRELLIS chains its stages differently from Hunyuan, so the extension needs support for it; the changes are covered under [Supporting TRELLIS.2](#6-supporting-trellis2). With that in place, keep **Hunyuan** selected as the engine in the Image to 3D panel. That choice routes generation to the Tiiny device, which runs whichever 3D model is loaded.

From the cut-out cat at High quality with 40,000 faces, my measured runs were:

| Run | Stages | Time | Result |
|---|---|---|---|
| Textured, 1024 px | shape, simplify, texture, export | 6.6 min | Solid painted cat, about 34,000 vertices |
| Untextured | shape, simplify | 4.3 min | Clean cat-shaped mesh, about 18,000 vertices |

Background removal shares the missing BriefNet model, so it fails with TRELLIS too. Until the device is fixed, feed it a PNG with a transparent background and leave **Remove Background on Tiiny** off.

### Two traps along the way

- **Restart Blender after changing the extension.** Saving new files into the extension folder doesn't change the copy Blender already has in memory. Quit with ⌘Q and reopen; closing the window is not enough. My first two attempts after the change failed because Blender was still running the old code.
- **Don't click Generate again after an HTTP 502.** A 502 on upload can arrive after the device has already accepted the task. A second click then gets HTTP 429 ("only one request may execute at a time"). Wait a few minutes for the first task to finish.

## Adapting the Windows-only extension for macOS

The published Image to 3D 1.1.5 declares `platforms = ["windows-x64"]`. Its engines (TRELLIS.2, Pixal3D, Hi3D, Klein, and local Hunyuan3D) run on an NVIDIA GPU through CUDA, which Macs do not have. The port does not try to run those engines on Apple silicon. Instead it keeps all of the add-on's user interface and import code, and sends Hunyuan jobs to the Tiiny device over the network.

The add-on is licensed GPL-3.0-or-later, so you may modify it for your own use. If you redistribute a modified build, the GPL's source and license terms apply.

Keep an untouched copy of the original folder before you start. I kept mine as `image_to_3d-1.1.5.bak_orig` and compared against it throughout.

### 1. Declare macOS in the manifest

In `blender_manifest.toml`, add the platform and describe the new network use:

```toml
# macos-arm64: Hunyuan3D-2.1 runs on a Tiiny AI device on the local network;
# the local CUDA engines (TRELLIS.2, Pixal3D, Hi3D, Klein) are Windows only
platforms = ["windows-x64", "macos-arm64"]

[permissions]
network = "Download AI models, connect to your Tiiny AI device"
files = "Read pictures, install the engine and models"
```

The original files use Windows (CRLF) line endings. Converting them to LF is harmless, but it makes a plain `diff` against the original show every line as changed. Use `diff --strip-trailing-cr` to see the real edits.

### 2. Move Windows-only calls behind one helper

The add-on called Windows APIs directly in several places. I gathered them into a new `platform_util.py` so the rest of the code loads and runs on a Mac:

| Windows original | Helper | macOS behaviour |
|---|---|---|
| `env\Scripts\python.exe` | `venv_python(env_dir)` | `env/bin/python` |
| `taskkill /PID … /T /F` | `kill_tree(p)` | `SIGTERM` to the process group, then `SIGKILL` |
| `creationflags=0x08000000` (no console window) | `NOWINDOW` | `0`; POSIX rejects that flag |
| Windows memory query | `mem_gb()` | `sysctl hw.memsize` and `vm_stat` |

Then replace each direct call in `__init__.py` with the helper, for example `platform_util.venv_python(...)` and `platform_util.kill_tree(p)`.

### 3. Decide which backend runs each engine

A single function answers, for each engine, whether it runs locally, on the Tiiny device, or not at all on this machine:

```python
def backend_for(engine):
    """'TIINY', 'LOCAL', or None when this machine cannot run the engine."""
    if engine == "HUNYUAN" and tiiny_on():
        return "TIINY"
    if IS_MAC:
        return None          # TRELLIS.2 / Pixal3D / Hi3D / Klein need CUDA
    return "LOCAL"
```

`engine_ready()`, the Generate button's `poll()`, and the panel in `card.py` all use it. On a Mac, CUDA-only engines show "needs an NVIDIA GPU (Windows)" with a hint to choose Hunyuan. A Tiiny engine without settings shows **Set Up Tiiny**. Features that only exist locally, such as the Hi3D refinement chain, multi-view input, and auto-enhance, are switched off for Tiiny jobs.

### 4. Make the network backend look like the local engine

Locally, the add-on starts a Python subprocess and reads lines like `PROGRESS 0.4 Building shape` and `DONE <path>` from its output. The new `tiiny_backend.py` provides a `TiinyProcess` class with the same `poll()`, `kill()`, `stdin`, and `stdout` surface as `subprocess.Popen`. It speaks the same line protocol, but runs the device's stage chain on a worker thread. Because of this, the existing reader, progress bar, cancel button, and GLB import code needed no changes; the add-on only chooses which "process" to start.

Some details that mattered:

- **Network calls never run on Blender's main thread.** A frozen settings snapshot is taken on the main thread when the job starts.
- **Status polls are retried on HTTP 502, 503, and 504** until the job timeout. Submissions are never retried after an ambiguous failure, because the device may already be running the task.
- **The API key is never sent to a different host.** The extension refuses redirects to another host and warns about public addresses and plain HTTP.
- **Cancel stops the worker** and asks the device to cancel the running task.

### 5. Add the Tiiny settings

New add-on preferences hold the base URL, API key source, authentication header, background-removal toggle, job timeout, and an optional CA file for HTTPS. **Use Tiiny AI** is on by default on macOS. The key can live in Blender's preferences (plain text on disk), the macOS login Keychain via `security add-generic-password`, or the `IMG3D_TIINY_API_KEY` environment variable. **Test Connection** checks health, the key, and the running model, and pairs with the device it found, so a different device answering at the same address is refused.

### 6. Supporting TRELLIS.2

The Tiiny SDK documentation describes Hunyuan's chain, where each stage's output file is uploaded again as the next stage's input. TRELLIS.2 on server 2.0.0 works differently. I found the details from `GET /v1/3d/capabilities` and the server's validation errors:

| Stage | TRELLIS.2 input | Output |
|---|---|---|
| `shape` | `image` upload | `trellis2_shape.zip` and a preview GLB |
| `simplify` | `shape_artifact` | `trellis2_geometry.zip` and a preview GLB |
| `texture` | `shape_artifact` | `trellis2_texture.zip` |
| `export` | `shape_artifact`, `material_artifact`, optional `geometry_artifact` | `trellis2.glb` |

The capabilities report `"input_refs": true`. Instead of uploading the 98 MB shape archive again, a stage can point at an artifact already on the device with a JSON form field:

```sh
curl -X POST "$BASE_URL/3d/tasks" \
  -H "Authorization: Bearer $TIINY_API_KEY" \
  -F stage=texture \
  -F 'params={"inference_quality":"balance"}' \
  -F 'input_refs={"shape_artifact":{"task_id":"<shape task_id>","artifact_id":"<shape artifact_id>"}}'
```

The extension reads the capabilities at the start of each job. If the texture stage takes a `shape_artifact`, it runs the TRELLIS chain with references; otherwise it runs the Hunyuan chain unchanged, so it switches back once Hunyuan is fixed. Two smaller changes went with it. Stage settings are filtered against each stage's `stage_specs.controls` as well as its `params`. **Test Connection** accepts a running TRELLIS model instead of insisting on Hunyuan.

### 7. Build and install

Build platform-specific packages from the extension folder with Blender's command-line tool:

```sh
cd image_to_3d-1.1.5
/Applications/Blender.app/Contents/MacOS/Blender --command extension build \
  --split-platforms --output-dir ../dist_build
```

`--split-platforms` writes one ZIP per declared platform, such as `image_to_3d-1.1.5-macos_arm64.zip`. Install it with **Edit → Preferences → Get Extensions → Install from Disk**, then restart Blender. When you edit an installed copy directly, also copy the changes back into your source folder. Otherwise the next build silently drops them.

## Source notes

This guide is based on the local Tiiny-enabled Image to 3D 1.1.5 extension’s manifest, preferences, and backend implementation, compared against the original 1.1.5 package; the Tiiny SDK Image-to-3D documentation preserved in the local wiki; and live device checks on November 7, 2026. The verified fixes were model loading (HTTP 503 changing to HTTP 200) and, for flat meshes, switching to TRELLIS.2-4B with the extension changes above. The vertex counts and timings come from runs through the updated extension, and the result was confirmed in the Blender interface. The extension adaptation is separate from the original add-on’s release.
