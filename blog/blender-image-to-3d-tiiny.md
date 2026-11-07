# Image to 3D in Blender with Tiiny: Setup and the HTTP 503 Fix

By Tae Lee · November 7, 2026 · 6 min read

Blender’s Image to 3D add-on can send a reference image to a Tiiny device running Hunyuan3D-2.1 and bring the generated mesh back into your scene. The setup has two parts: configure Blender’s connection, and load the 3D model on the device. A downloaded model alone is not enough.

These notes cover my locally adapted **Image to 3D 1.1.5** extension with a Tiiny backend. The stock add-on may not include these settings. This post does not distribute the adapted extension; you need a Tiiny-enabled package before following the Blender steps.

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
5. Leave **Remove Background on Tiiny** enabled for an ordinary photo. You can turn it off for an image already cut out.
6. If Blender reports network access is disabled, enable **Preferences → System → Allow Online Access**.
7. Click **Test Connection** and wait for the result. Save your preferences if automatic saving is disabled.

For macOS Keychain storage, use the key-storage button shown after selecting that option. Preferences storage masks the key on screen but stores it as plain text in Blender’s preferences. The environment-variable option reads `IMG3D_TIINY_API_KEY` from Blender’s process environment; set it before launching Blender.

## 4. Generate your first mesh

Open the add-on’s **Image to 3D** viewport interface, select **Hunyuan**, and choose a reference image. Start with a modest quality setting and click **Generate 3D**. Watch the stage and progress display while the device works.

The API pipeline consists of background removal, shape generation, optional simplification, texturing, and export. The adapted backend downloads the resulting GLB and imports it into Blender. Inspect the geometry, materials, orientation, and scale before using the asset in a larger scene. Save the Blender project once you have a result you want to keep.

The connection and model-loading checks below were verified against the physical device. A complete image-to-mesh generation was not measured in this troubleshooting session, so these notes make no generation-time or output-quality claim.

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

## Source notes

This guide is based on the local Tiiny-enabled Image to 3D 1.1.5 extension’s manifest, preferences, and backend implementation; the Tiiny SDK Image-to-3D documentation preserved in the local wiki; and live device checks on November 7, 2026. The verified fix was model loading and HTTP 503 changing to HTTP 200. The extension adaptation is separate from the original add-on’s release.
