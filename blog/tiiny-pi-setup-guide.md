# Run Pi on Omnigent with Qwen3.6-35B-A3B on a Tiiny Pocket

By Tae Lee · November 7, 2026 · 8 min read

A Tiiny Pocket serves its models through an OpenAI-compatible Chat Completions API. Pi, the coding agent harness that Omnigent runs, can use any such endpoint as a custom provider. This post shows how to point Pi at **Qwen/Qwen3.6-35B-A3B** on the device, with one entry in `~/.pi/agent/models.json`.

Expect plain chat to work. Check tool calling, context size, and throughput on your own device before you rely on them; see [Known limitations](#known-limitations).

## What you need

- A Tiiny Pocket that offers Qwen3.6-35B-A3B in its Model Store, downloaded and loaded.
- The device's OpenAI base URL, its API key, and the exact model ID it serves.
- Pi installed on the machine that runs Omnigent, with Node.js 22.19 or later.
- A running Omnigent server. The default address is `http://127.0.0.1:6767`.

## 1. Collect the device details

| Item | What it is | Example |
|------|------------|---------|
| Base URL | The OpenAI API base, including `/v1` | `http://192.168.1.100:8000/v1` |
| API key | The key TiinyOS issues | Keep it private |
| Model ID | The exact ID the device serves | `Qwen/Qwen3.6-35B-A3B` |
| Context limit | The largest context the device accepts | Check TiinyOS |

TiinyOS shows the address and API key in its settings, and the model under the Model Store. Keep the key out of screenshots and shared files.

Check that the device answers before you touch Pi:

```bash
curl -H "Authorization: Bearer $TIINY_API_KEY" http://DEVICE_HOST:PORT/v1/models
```

## 2. Add a Tiiny provider to models.json

Pi reads custom providers from `~/.pi/agent/models.json`. A provider entry takes these fields:

| Field | Required | Meaning |
|-------|----------|---------|
| `baseUrl` | Yes | The device's OpenAI base URL, including `/v1` |
| `api` | Yes | `"openai-completions"` for Chat Completions |
| `apiKey` | Yes | A literal key, `"$NAME"` to read an environment variable, or `"!command"` to use a command's output |
| `authHeader` | No (default `false`) | `true` sends the key as `Authorization: Bearer` |
| `models` | Yes | The models this provider serves |

Each model needs an `id` that exactly matches the device's model ID. `name` sets the label in Pi, and `input` defaults to `["text"]`.

Read the key from an environment variable rather than writing it into the file:

```json
{
  "$schema": "https://pi.dev/schemas/models.schema.json",
  "providers": {
    "tiiny": {
      "name": "Tiiny Pocket",
      "baseUrl": "http://DEVICE_HOST:PORT/v1",
      "api": "openai-completions",
      "apiKey": "$TIINY_API_KEY",
      "authHeader": true,
      "models": [
        {
          "id": "Qwen/Qwen3.6-35B-A3B",
          "name": "Tiiny Qwen3.6-35B-A3B",
          "input": ["text"]
        }
      ]
    }
  }
}
```

Replace `DEVICE_HOST:PORT` with your device's address, then export the key in the shell that starts Omnigent, or in your shell profile:

```bash
export TIINY_API_KEY="<your-tiiny-api-key>"
```

If the key lives in a secret manager, use a command instead:

```json
"apiKey": "!cat ~/.secrets/tiiny-key"
```

### Optional model fields

| Field | Default | Meaning |
|-------|---------|---------|
| `contextWindow` | 128,000 | Context size in tokens. Set it to what the device actually accepts. |
| `maxTokens` | 16,384 | Maximum output tokens per turn. A smaller value suits a small device. |
| `reasoning` | `false` | Whether the model supports thinking |
| `compat` | none | API compatibility overrides |

For example, add `"contextWindow": 131072, "maxTokens": 8192` to the model entry once you have confirmed those limits on your device.

## 3. Reload and check the provider

Pi reloads `models.json` when you open the model picker with `/model`. Some versions also have `/reload`. From a terminal, list the models Pi knows about:

```bash
pi --list-models
```

The Tiiny provider and `Qwen/Qwen3.6-35B-A3B` should be in the list. If they are missing, see [Troubleshooting](#troubleshooting).

## 4. Pick the model in Omnigent

Start a new Pi session in the Omnigent web UI and select the model:

```
/model Qwen/Qwen3.6-35B-A3B
```

Then send a short message to confirm the connection.

An Omnigent agent spec does not need its own Tiiny setting. The Omnigent runner hands the session to Pi, and Pi reads the provider from `models.json`.

## 5. Test it end to end

| Test | How | Expected |
|------|-----|----------|
| Basic chat | Send a text message | A reply arrives |
| Tool call | Ask it to read a file or run a shell command | A structured tool call runs |
| Tool result | Look at the next turn | The model uses the tool's output |
| Structure | Watch how tools are invoked | Real tool-call blocks, not a prose description of a call |

The tool tests matter most for agent work. If the model only describes what it would do, the variant you loaded may not support tool calling.

## Troubleshooting

### The model does not appear in Pi

1. Check that `~/.pi/agent/models.json` is valid JSON.
2. Make sure Pi is reading that file and not a different agent directory.
3. Run `/model` again to reload.
4. Check the device: `curl http://DEVICE_HOST:PORT/v1/models`.

### Authentication errors

- Confirm the key is current, and that `TIINY_API_KEY` is set in the environment of the process that runs Pi.
- Set `"authHeader": true` if the device expects a bearer token.
- Call the device directly with the same key:

```bash
curl http://DEVICE_HOST:PORT/v1/chat/completions \
  -H "Authorization: Bearer $TIINY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3.6-35B-A3B", "messages": [{"role": "user", "content": "hi"}]}'
```

### "Invalid request" errors

These usually mean one of the following:

- The base URL is wrong, for example missing `/v1` or using the wrong port.
- The device rejected the key.
- The model ID does not match what the device serves.

### Errors inside an Omnigent session

1. Check the Pi runner logs for provider errors.
2. Confirm the Omnigent host can reach the device over the network.
3. Compare the model ID with the device's `/v1/models` output.

## Known limitations

1. **Text only.** Use `input: ["text"]`. Image input has not been confirmed.
2. **Tool calling varies.** It depends on the model variant the device loads. Run the tests in step 5 before you give the agent real work.
3. **Device serving limits.** Rate limits, context window, and throughput depend on the device. Confirm them in TiinyOS.
4. **Reasoning settings.** Do not assume Pi's thinking settings control Qwen's reasoning on Tiiny. No mapping between them has been established.
5. **Keys are read once.** Pi reads the bearer key when the session starts. A short-lived key can expire during a long session.

Start with one agent, short tasks, and modest context and output limits. Raise them once you know what your device handles.

## Source notes

This guide comes from my Omnigent knowledge-base notes on configuring Pi with a Tiiny provider, and from Pi's [models configuration](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md) and [custom provider](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/custom-provider.md) documentation. The `apiKey` forms, `authHeader`, and the model-field defaults are taken from Pi's documentation. Tool-calling and streaming behaviour on the device remain to be confirmed.
