# NemoClaw + Omni VLM Sub-Agent Demo

Add a vision-capable sub-agent to OpenClaw in a NemoClaw sandbox using **Nemotron-3 Nano Omni Reasoning 30B**.

## Architecture

```
User → Main Agent (Nemotron Super 120B, text-only)
           │ sessions_spawn
           ▼
       Vision-Operator (Nemotron Omni 30B, text + image)
           │ read tool on image
           ▼
       Image analysis returned to main agent → user
```

The main agent handles conversation via the Privacy Router. When it encounters an image task, it delegates to a vision-operator sub-agent that calls the NVIDIA API directly using the Omni model.

## Getting Started

Follow the step-by-step guide: **[vlm-subagent-guide.md](vlm-subagent-guide.md)**

## What's Included

| File | Purpose |
|------|---------|
| [`vlm-subagent-guide.md`](vlm-subagent-guide.md) | Complete setup walkthrough |
| [`vlm-subagent/openclaw-patch.py`](vlm-subagent/openclaw-patch.py) | Script to patch openclaw.json with Omni provider + agents |
| [`vlm-subagent/TOOLS.md`](vlm-subagent/TOOLS.md) | Agent instructions (uploaded into sandbox workspace) |
| [`vlm-subagent/openclaw-reference.json`](vlm-subagent/openclaw-reference.json) | Reference patched config for comparison |
| [`vlm-subagent/auth-profiles.template.json`](vlm-subagent/auth-profiles.template.json) | Template for vision-operator auth |

## Prerequisites

- A running NemoClaw sandbox with OpenClaw (Nemotron Super 120B)
- NVIDIA API key (`nvapi-...`) with access to the Omni model

## Based On

[Haran Kumar's nemoclaw-with-omni-subagent](https://gitlab-master.nvidia.com/hshivkumar/nemoclaw-with-omni-subagent)
