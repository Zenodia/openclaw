---
summary: "Use NVIDIA's OpenAI-compatible API in OpenClaw"
read_when:
  - You want to use NVIDIA models in OpenClaw
  - You need NVIDIA_API_KEY setup
title: "NVIDIA"
---

# NVIDIA

NVIDIA provides an OpenAI-compatible API at `https://integrate.api.nvidia.com/v1` for Nemotron and NeMo models. Authenticate with an API key from [NVIDIA NGC](https://catalog.ngc.nvidia.com/).

## CLI setup

Export the key once, then run onboarding and set an NVIDIA model:

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/llama-3.1-nemotron-70b-instruct
```

If you still pass `--token`, remember it lands in shell history and `ps` output; prefer the env var when possible.

## Config snippet

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/llama-3.1-nemotron-70b-instruct" },
    },
  },
}
```

## Model IDs

NVIDIA NIM uses an `org/model-name` format. When referencing a model in OpenClaw config or CLI, prefix the full model ID with the provider name:

| NVIDIA NIM model ID                           | OpenClaw reference                                   |
| --------------------------------------------- | ---------------------------------------------------- |
| `nvidia/llama-3.1-nemotron-70b-instruct`      | `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`      |
| `nvidia/llama-3.3-nemotron-super-49b-v1.5`    | `nvidia/nvidia/llama-3.3-nemotron-super-49b-v1.5`    |
| `meta/llama-3.3-70b-instruct`                 | `nvidia/meta/llama-3.3-70b-instruct`                 |
| `nvidia/mistral-nemo-minitron-8b-8k-instruct` | `nvidia/nvidia/mistral-nemo-minitron-8b-8k-instruct` |

The format is always `{provider}/{model-id}`. For NVIDIA-owned models, this results in a double `nvidia/` prefix.

## Notes

- OpenAI-compatible `/v1` endpoint; use an API key from NVIDIA NGC.
- Provider auto-enables when `NVIDIA_API_KEY` is set; uses static defaults (131,072-token context window, 4,096 max tokens).
- `nvidia/llama-3.3-nemotron-super-49b-v1.5` is a reasoning model (extended thinking supported).
