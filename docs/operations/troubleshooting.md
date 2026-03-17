---
tags:
  - operations
  - troubleshooting
---

# Troubleshooting

## First steps

Before diving into specific issues:

```bash
# Run the health check
missy doctor

# Check provider availability
missy providers

# Review recent errors in the audit log
missy audit recent --limit 20
```

## Provider issues

### "Provider not available"

**Symptoms:** `missy providers` shows a provider as unavailable.

**Causes and fixes:**

=== "Anthropic"

    1. Check that the API key is set:
    ```bash
    echo $ANTHROPIC_API_KEY
    ```
    2. Verify the key format -- must start with `sk-ant-api` (not `sk-ant-oat`, which is a setup token).
    3. Check network policy allows `api.anthropic.com`:
    ```yaml
    network:
      provider_allowed_hosts:
        - "api.anthropic.com"
    ```

=== "OpenAI"

    1. Check the API key:
    ```bash
    echo $OPENAI_API_KEY
    ```
    2. Verify network policy allows `api.openai.com`.

=== "Ollama"

    1. Verify Ollama is running:
    ```bash
    curl http://localhost:11434/api/tags
    ```
    2. Check the model is pulled:
    ```bash
    ollama list
    ```
    3. If using a remote Ollama server, verify `base_url` in config.

### "Request timeout"

Increase the timeout in provider config:

```yaml
providers:
  ollama:
    timeout: 120    # Default is 30
```

Large local models (70B+) may need 60-120 seconds for first inference.

### Rate limit errors

If you hit rate limits:

1. Configure API key rotation with multiple keys:
```yaml
providers:
  anthropic:
    api_keys:
      - "sk-ant-key1..."
      - "sk-ant-key2..."
```
2. Use the `ModelRouter` to route simple queries to cheaper/faster models.

## Policy denials

### "Policy denied execution of tool"

The policy engine blocked a tool because the required permission is not granted.

Check the audit log for details:

```bash
missy audit security --limit 10
```

Common fixes:

=== "Shell tool denied"

    ```yaml
    shell:
      enabled: true
      allowed_commands:
        - "ls"
        - "cat"
        - "grep"
    ```

=== "File read denied"

    ```yaml
    filesystem:
      allowed_read_paths:
        - "/home/user/workspace"
    ```

=== "File write denied"

    ```yaml
    filesystem:
      allowed_write_paths:
        - "/home/user/workspace/output"
    ```

=== "Network denied"

    ```yaml
    network:
      tool_allowed_hosts:
        - "api.example.com"
    ```

### "Network policy denied"

All outbound HTTP passes through `PolicyHTTPClient`. If a request is blocked:

1. Check which host was denied in the audit log.
2. Add it to the appropriate allow list:

```yaml
network:
  allowed_domains:
    - "example.com"          # General
  provider_allowed_hosts:
    - "api.anthropic.com"    # Provider-specific
  tool_allowed_hosts:
    - "api.weather.com"      # Tool-specific
  discord_allowed_hosts:
    - "discord.com"          # Discord-specific
```

## Voice channel issues

### Edge node cannot connect

1. Verify the server is running:
```bash
missy gateway status
missy voice status
```

2. Check the server is listening on the correct interface:
```yaml
voice:
  host: "0.0.0.0"    # All interfaces (not 127.0.0.1)
  port: 8765
```

3. Test connectivity from the Pi:
```bash
curl -v ws://SERVER_IP:8765
```

4. Check firewall rules on the server.

### "auth_fail" in edge node logs

1. Verify the device is paired:
```bash
missy devices list
```

2. Check the token is correct and properly provisioned:
```bash
# On the Pi
sudo cat /etc/missy-edge/token
```

3. If the token is lost, re-pair:
```bash
missy devices unpair NODE_ID
# Then re-pair (see Pairing guide)
```

### No audio response (STT/TTS issues)

1. Check STT engine is loaded:
```bash
missy voice status
```

2. Verify faster-whisper is installed:
```bash
pip list | grep faster-whisper
```

3. Verify Piper binary is available:
```bash
which piper
piper --version
```

4. Check the voice model exists:
```bash
ls ~/.local/share/piper/
```

### Wake word not triggering

See the [Wake Word troubleshooting section](../edge/wake-word.md#troubleshooting).

## Discord issues

### Bot not responding

1. Check the bot token is valid:
```bash
missy discord probe
```

2. Verify Discord is enabled in config:
```yaml
discord:
  enabled: true
```

3. Check guild policies allow the channel:
```yaml
guild_policies:
  "GUILD_ID":
    enabled: true
    allowed_channels: []    # Empty = all channels
```

4. Check network policy allows Discord hosts:
```bash
missy audit security --limit 10
```

### Slash commands not showing

1. Register commands:
```bash
missy discord register-commands --guild-id YOUR_GUILD_ID
```

2. For global commands, wait up to 1 hour for propagation.

3. Verify the bot has `applications.commands` scope in the invite URL.

## Config issues

### Config not loading

1. Validate YAML syntax:
```bash
python3 -c "import yaml; yaml.safe_load(open('$HOME/.missy/config.yaml'))"
```

2. Check file permissions:
```bash
ls -la ~/.missy/config.yaml
```

3. Roll back to a working config:
```bash
missy config rollback
```

### Hot-reload not working

The `ConfigWatcher` polls the config file every 1 second with a 2-second debounce. If changes are not being picked up:

1. Verify the file was actually modified (check `stat`).
2. Check for YAML syntax errors in the updated file.
3. Review logs at `debug` level for reload messages.

## Memory and performance

### High memory usage

1. Clean up old sessions:
```bash
missy sessions cleanup --older-than 7
```

2. Reduce the context window:
```yaml
# The default token budget is 30,000 tokens
# Reducing it lowers memory usage
```

### Slow responses

1. Check if you are hitting the circuit breaker:
```bash
missy audit recent --limit 10
```
The circuit breaker opens after 5 consecutive failures with exponential backoff (60s base, 300s max).

2. Switch to a faster provider or model:
```bash
missy ask "quick question" --provider ollama
```

## Getting help

If you are stuck:

1. Run `missy doctor` and note any failures.
2. Check `~/.missy/audit.jsonl` for recent error events.
3. Set `log_level: "debug"` temporarily for detailed logs.
4. Open an issue at [github.com/MissyLabs/missy/issues](https://github.com/MissyLabs/missy/issues) with the doctor output and relevant audit log entries.
