# Adding a Vision (VLM) Sub-Agent to OpenClaw in a NemoClaw Sandbox

This guide walks you through adding a vision-capable sub-agent to OpenClaw running inside an OpenShell sandbox. By the end, your AI agent will be able to analyze images by delegating to a **Nemotron-3 Nano Omni Reasoning 30B** vision model.

The setup creates a two-agent system:
- **Main agent** (Nemotron Super 120B, text-only) handles conversation and delegates image tasks
- **Vision-operator sub-agent** (Nemotron Omni 30B, vision-capable) analyzes images and returns results

The main agent uses the Privacy Router at `inference.local`. The vision-operator bypasses the Privacy Router and calls the NVIDIA API directly, since the Privacy Router only serves a single text-only model.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Running OpenClaw sandbox | A working OpenShell sandbox with OpenClaw (Nemotron Super 120B via Privacy Router). |
| NVIDIA API key | An API key (starts with `nvapi-`) with access to the Omni model. |

## Part 1: Set Variables

Everything below uses these — set them once:

``` bash
SANDBOX=<your-sandbox-name>
DOCKER_CTR=openshell-cluster-nemoclaw
source ~/.env   # or: export NVIDIA_API_KEY=nvapi-...
```

## Part 2: Update the OpenShell Network Policy

The OpenClaw gateway runs as `/usr/local/bin/node`. The default sandbox policy allows `integrate.api.nvidia.com` for `claude` and `openclaw` binaries, but **not** for `node`. Without this change, the vision-operator will fail with `LLM request timed out.` errors.

### 2a. Export the current policy

``` bash
openshell policy get $SANDBOX --full > /tmp/raw-policy.txt
```

The output includes 7 lines of metadata header. Strip them to get clean YAML:

``` bash
sed -n '8,$p' /tmp/raw-policy.txt > /tmp/current-policy.yaml
```

### 2b. Add `/usr/local/bin/node` to the `nvidia` policy block

Find the `nvidia:` section and add `node` to its `binaries` list. Before:

``` yaml
    binaries:
    - path: /usr/local/bin/claude
    - path: /usr/local/bin/openclaw
```

After:

``` yaml
    binaries:
    - path: /usr/local/bin/claude
    - path: /usr/local/bin/openclaw
    - path: /usr/local/bin/node
```

### 2c. Apply the updated policy

``` bash
openshell policy set --policy /tmp/current-policy.yaml $SANDBOX
```

You should see:

```
✓ Policy version N submitted (hash: ...)
```

Verify:

``` bash
openshell policy get $SANDBOX --full | grep -A 5 "nvidia:" | grep node
```

## Part 3: Patch openclaw.json

The sandbox config only has the main inference provider. We need to add:
- `nvidia-omni` provider pointing directly at the NVIDIA API
- `agents.list` defining `main` and `vision-operator`
- `agents.defaults.timeoutSeconds: 300` to prevent sub-agent announce timeouts

### 3a. Fetch the current config

``` bash
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- cat /sandbox/.openclaw/openclaw.json > /tmp/remote_openclaw.json
```

### 3b. Run the patch script

The patch script is at [`vlm-subagent/openclaw-patch.py`](vlm-subagent/openclaw-patch.py) in this repository. Copy it to `/tmp` and run it:

``` bash
python3 /tmp/openclaw-patch.py "$NVIDIA_API_KEY" \
  < /tmp/remote_openclaw.json > /tmp/updated_openclaw.json
```

> You can compare the result against [`vlm-subagent/openclaw-reference.json`](vlm-subagent/openclaw-reference.json) if you want to verify the patch output.

### 3c. Push the patched config to the sandbox

``` bash
# Unlock config + hash
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- chmod 644 /sandbox/.openclaw/openclaw.json
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- chmod 644 /sandbox/.openclaw/.config-hash

# Write patched config
cat /tmp/updated_openclaw.json | docker exec -i $DOCKER_CTR \
  kubectl exec -i -n openshell $SANDBOX -c agent \
  -- sh -c 'cat > /sandbox/.openclaw/openclaw.json'

# Regenerate the integrity hash
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- /bin/bash -c "cd /sandbox/.openclaw && sha256sum openclaw.json > .config-hash"

# Lock everything back
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- chmod 444 /sandbox/.openclaw/openclaw.json
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- chmod 444 /sandbox/.openclaw/.config-hash
```

## Part 4: Create Auth Profiles for the Vision-Operator

The gateway strips API keys from `openclaw.json` when creating per-agent configs. The vision-operator calls the NVIDIA API directly and needs its own auth store. The main agent doesn't need one — it uses the Privacy Router.

``` bash
# Create the auth profile
cat > /tmp/auth-profiles.json << EOF
{
  "providers": {
    "nvidia-omni": {
      "apiKey": "$NVIDIA_API_KEY"
    }
  }
}
EOF

# Write it to the vision-operator's agent directory
cat /tmp/auth-profiles.json | docker exec -i $DOCKER_CTR \
  kubectl exec -i -n openshell $SANDBOX -c agent \
  -- sh -c 'cat > /sandbox/.openclaw-data/agents/vision-operator/agent/auth-profiles.json'

# IMPORTANT: Fix ownership so the gateway (runs as sandbox user) can write sessions
docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent \
  -- chown -R sandbox:sandbox /sandbox/.openclaw-data/agents/vision-operator
```

> A template is also available at [`vlm-subagent/auth-profiles.template.json`](vlm-subagent/auth-profiles.template.json).

If you skip this step, the gateway will log `No API key found for provider "nvidia-omni"` and fall back to the text-only model, producing hallucinated image descriptions.

If you skip the `chown`, the gateway will fail with `EACCES: permission denied, mkdir '/sandbox/.openclaw/agents/vision-operator/sessions'` when the main agent tries to spawn the vision-operator.

## Part 5: Upload TOOLS.md

TOOLS.md teaches both agents how to handle image tasks. The main agent learns it must delegate via `sessions_spawn`, and the vision-operator learns it can use `read` directly on images.

The file is at [`vlm-subagent/TOOLS.md`](vlm-subagent/TOOLS.md) in this repository. Upload it to the sandbox workspace:

``` bash
cat /path/to/TOOLS.md | docker exec -i $DOCKER_CTR \
  kubectl exec -i -n openshell $SANDBOX -c agent \
  -- sh -c 'cat > /sandbox/.openclaw-data/workspace/TOOLS.md'
```

## Part 6: Upload Test Images and Verify

Upload images to the workspace directory (NOT `/sandbox/` root):

``` bash
openshell sandbox upload $SANDBOX ~/my-test-image.jpg /sandbox/.openclaw-data/workspace/
```

Then open the TUI and test:

``` bash
openclaw tui
```

## Trying It Out

Ask the agent to analyze an image:

- "Describe the image frame_000270.jpg in the workspace and write the description to image-description.md"
- "What objects are visible in /sandbox/.openclaw-data/workspace/test.jpg?"
- "Read the text in /sandbox/.openclaw-data/workspace/document.png"

The main agent should call `sessions_spawn` with `agentId: "vision-operator"` rather than trying to read the image itself. The vision-operator will analyze the image and return the result through the sessions system.

## How It All Fits Together

```
┌─────────────────────────────────────────────────────┐
│  User sends message with image task                 │
│                        │                            │
│                        ▼                            │
│  ┌──────────────────────────────┐                   │
│  │  Main Agent                  │                   │
│  │  Nemotron Super 120B         │                   │
│  │  (text-only)                 │                   │
│  │  via Privacy Router          │                   │
│  │  (inference.local)           │                   │
│  └──────────┬───────────────────┘                   │
│             │ sessions_spawn                        │
│             ▼                                       │
│  ┌──────────────────────────────┐                   │
│  │  Vision-Operator Sub-Agent   │                   │
│  │  Nemotron Omni 30B           │                   │
│  │  (text + image)              │                   │
│  │  via NVIDIA API direct       │                   │
│  │  (integrate.api.nvidia.com)  │                   │
│  └──────────┬───────────────────┘                   │
│             │ read tool on image file               │
│             ▼                                       │
│  Image analysis returned to main agent → user       │
└─────────────────────────────────────────────────────┘
```

### Key paths

| Path | Purpose |
|------|---------|
| `/sandbox/.openclaw/openclaw.json` | Root-owned, read-only config |
| `/sandbox/.openclaw/.config-hash` | SHA256 integrity check |
| `/sandbox/.openclaw-data/workspace/` | Writable workspace (canonical path) |
| `/sandbox/.openclaw-data/agents/vision-operator/agent/auth-profiles.json` | Vision-operator API key |
| `/tmp/gateway.log` | Gateway stdout/stderr |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `LLM request timed out.` / `Connection error.` | The #1 issue. Check that `/usr/local/bin/node` is in the `nvidia` binaries list in the network policy. Redo Part 2. |
| `No API key found for provider "nvidia-omni"` | Vision-operator's `auth-profiles.json` is missing or wrong. Redo Part 4. |
| Main agent doesn't delegate to vision-operator | Verify `agents.list` in openclaw.json is correct and the gateway picked up the change. Try explicitly: "use the vision operator to analyze this image." |
| `Action send requires a target.` / `Unknown channel: webchat` | Vision-operator has `message` tool available. Ensure `"deny": ["message", "sessions_spawn"]` is set in its tools config. |
| Sub-agent announce timeout (60000ms) | `agents.defaults.timeoutSeconds` not set to 300. Re-run the patch script. |
| `EACCES: permission denied, mkdir .../vision-operator/sessions` | The vision-operator agent directory is owned by root. Run: `docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent -- chown -R sandbox:sandbox /sandbox/.openclaw-data/agents/vision-operator` |
| EISDIR error / wrong path | Agents using `/sandbox/.openclaw/workspace` (symlink) instead of `/sandbox/.openclaw-data/workspace/`. Verify TOOLS.md is present in the workspace. |
| Stale sessions / `session file locked` | Clear session data: `docker exec $DOCKER_CTR kubectl exec -n openshell $SANDBOX -c agent -- rm -rf /sandbox/.openclaw-data/agents/*/sessions/*` |

## Tailing Logs

From inside the sandbox:

``` bash
# Gateway log
tail -f /tmp/gateway.log

# Detailed JSON log
tail -f /tmp/openclaw/openclaw-$(date -u +%Y-%m-%d).log
```

## Starting Over

``` bash
nemoclaw $SANDBOX destroy --yes
nemoclaw onboard
# Repeat Parts 1–6
```

## Based On

This guide is based on [Haran Kumar's nemoclaw-with-omni-subagent](https://gitlab-master.nvidia.com/hshivkumar/nemoclaw-with-omni-subagent/-/tree/main?ref_type=heads), adapted into a step-by-step demo format.
