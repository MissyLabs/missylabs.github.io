---
tags:
  - getting-started
  - configuration
---

# Setup Wizard

The setup wizard creates `~/.missy/config.yaml` with your provider credentials and security policies. It supports both an interactive mode (the default) and a non-interactive mode for scripting and CI.

## Interactive mode

```bash
missy setup
```

### Step 1: Workspace

The wizard prompts for a workspace directory. This is where Missy is allowed to read and write files.

```
Step 1 of 5 — Workspace
  Workspace directory [~/workspace]:
```

Press Enter to accept the default (`~/workspace`) or type a custom path. The directory is created automatically if it does not exist.

### Step 2: Provider selection

```
Step 2 of 5 — AI Provider(s)
  Choose which providers to configure:

    1. Anthropic (Claude)
    2. OpenAI (GPT-4o / Codex)
    3. Ollama (local models)
    4. Anthropic + OpenAI (both)
    5. All three
    0. Skip (configure manually later)
```

You can configure multiple providers. The first one listed becomes the default.

### Step 3: API key and authentication

The wizard supports several authentication methods depending on the provider.

=== "Anthropic"

    Three authentication options:

    ```
    Auth method:
      1. API key  (sk-ant-api…)  [recommended]
      2. API key + vault  (encrypted, config references vault://)
      3. Claude Code setup-token  (sk-ant-oat…)  [ToS risk]
    ```

    **Option 1 (API key)** is the standard path. The wizard checks for the `ANTHROPIC_API_KEY` environment variable first:

    ```
    Detected ANTHROPIC_API_KEY in environment: sk-ant…wxyz
    Use this key? [Y/n]:
    ```

    If the variable is set and you confirm, Missy uses it at runtime without embedding the key in the config file.

    **Option 2 (vault)** encrypts the key using Missy's ChaCha20-Poly1305 vault and stores a `vault://ANTHROPIC_API_KEY` reference in the config.

    !!! warning "Option 3: setup-tokens"
        Setup tokens (`sk-ant-oat...`) are issued by Claude Code and are **not supported** by the Anthropic API. The wizard warns about Terms of Service implications. Use a regular API key from [console.anthropic.com](https://console.anthropic.com/settings/keys) instead.

=== "OpenAI"

    Two authentication options:

    ```
    Auth method:
      1. API key  (sk-…)
      2. OAuth / Codex CLI  (browser flow, PKCE)
    ```

    **Option 1** prompts for an API key, with environment variable detection for `OPENAI_API_KEY`.

    **Option 2** opens a browser-based OAuth flow (PKCE) for ChatGPT / Codex CLI access. This switches the provider to `openai-codex`, which routes through `chatgpt.com/backend-api` instead of `api.openai.com`.

    !!! tip "Remote / headless OAuth"
        For servers without a display, forward port 1455: `ssh -L 1455:localhost:1455 user@host`, then run the wizard.

=== "Ollama"

    No API key required. The wizard prompts for:

    - **Base URL** (default: `http://localhost:11434`)
    - **Model name** (default: `llama3`)

    Ollama runs locally, so no network policy entries are added.

After entering credentials, the wizard offers to verify connectivity with a test API call:

```
Verify API key with a test call? [Y/n]:
  Connecting…
  Connection successful.
```

### Model tiers

For Anthropic and OpenAI, the wizard presents the available models and asks you to assign three tiers:

```
Available models:
  1. Sonnet 4.6 — balanced (recommended)  (claude-sonnet-4-6)
  2. Haiku 4.5 — fastest / cheapest  (claude-haiku-4-5-20251001)
  3. Opus 4.6 — most capable  (claude-opus-4-6)

  Primary model [1-3]: 1
  Fast (simple tasks) model [1-3]: 2
  Premium (complex tasks) model [1-3]: 3
```

| Tier | Used for |
|------|----------|
| **Primary** | Default model for `missy ask` and `missy run` |
| **Fast** | Simple tasks, internal lookups, cost-sensitive operations |
| **Premium** | Complex multi-step reasoning, code generation |

### Step 4: Discord (optional)

```
Step 4 of 5 — Discord Integration  (optional)
  Configure Discord bot? [y/N]:
```

If you choose yes, the wizard collects:

- **Bot token** (hidden input, from discord.com/developers)
- **Application ID**
- **DM policy**: disabled, allowlist, pairing, or open
- **Guild policies**: per-server settings for require-mention, allowed channels, and capability mode
- **Acknowledgement reaction** (default: `eyes`)
- **Ignore bots** toggle

!!! note
    Discord setup adds `discord.com` and `gateway.discord.gg` to the network allowlist automatically.

### Step 5: Review and write

The wizard displays a summary table and asks for confirmation:

```
┏━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Setting            ┃ Value                                     ┃
┡━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Config path        │ /home/you/.missy/config.yaml              │
│ Workspace          │ /home/you/workspace                       │
│ Provider: anthropic│ claude-sonnet-4-6  auth=sk-ant…wxyz       │
│ Verified: anthropic│ OK                                        │
└────────────────────┴───────────────────────────────────────────-┘

  Write this configuration? [Y/n]:
```

On confirmation, the wizard:

1. Backs up any existing config file
2. Writes `~/.missy/config.yaml` atomically (permissions: `0600`)
3. Creates `~/.missy/secrets/`, `~/.missy/logs/`, and `~/.missy/jobs.json` if missing

---

## Non-interactive mode

For automated deployments, CI pipelines, or scripted setups, pass `--no-prompt` with the required options:

```bash
missy setup \
  --provider anthropic \
  --api-key-env ANTHROPIC_API_KEY \
  --no-prompt
```

### Required flags

| Flag | Description |
|------|-------------|
| `--provider NAME` | Provider name: `anthropic`, `openai`, `openai-codex`, or `ollama` |
| `--no-prompt` | Disable all interactive prompts |

### Authentication (one of)

| Flag | Description |
|------|-------------|
| `--api-key-env VAR` | Name of environment variable containing the API key |
| `--api-key VALUE` | API key value directly (use with caution) |

!!! warning "Avoid `--api-key` in scripts"
    Passing an API key on the command line exposes it in shell history and process listings. Prefer `--api-key-env` and set the variable in your environment.

### Optional flags

| Flag | Description | Default |
|------|-------------|---------|
| `--model NAME` | Model identifier | Provider's primary model |
| `--workspace PATH` | Workspace directory | `~/workspace` |

### Examples

=== "Anthropic from env var"

    ```bash
    export ANTHROPIC_API_KEY="sk-ant-api03-..."
    missy setup --provider anthropic --api-key-env ANTHROPIC_API_KEY --no-prompt
    ```

=== "OpenAI with custom model"

    ```bash
    export OPENAI_API_KEY="sk-..."
    missy setup \
      --provider openai \
      --api-key-env OPENAI_API_KEY \
      --model gpt-4-turbo \
      --no-prompt
    ```

=== "Ollama (no key needed)"

    ```bash
    missy setup --provider ollama --model llama3 --no-prompt
    ```

=== "CI / Docker"

    ```dockerfile
    RUN pip install -e . && \
        missy setup \
          --provider anthropic \
          --api-key-env ANTHROPIC_API_KEY \
          --workspace /app/workspace \
          --no-prompt
    ```

The non-interactive mode writes the same `config.yaml` structure as the interactive wizard, with identical security defaults (network deny-all, shell disabled, plugins disabled).

---

## Re-running the wizard

You can run `missy setup` again at any time. If a config file already exists, the wizard asks before overwriting and creates a timestamped backup of the previous config.

!!! tip "Editing config directly"
    After initial setup, most users edit `~/.missy/config.yaml` directly. Missy supports hot-reload: changes take effect without restarting. See [Configuration Reference](../configuration/reference.md) for the full schema.
