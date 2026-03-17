---
tags:
  - getting-started
  - installation
---

# Installation

## System requirements

| Requirement | Minimum |
|-------------|---------|
| **OS** | Linux (x86_64 or aarch64) |
| **Python** | 3.11 or later |
| **pip** | 21.0+ (for editable installs) |
| **Disk** | ~50 MB for Missy + dependencies |

!!! note "Linux only"
    Missy targets Linux as a first-class platform. macOS may work for development but is not tested or supported. Windows is not supported.

Check your Python version:

```bash
python3 --version  # (1)!
```

1. Must print `Python 3.11.x` or later. On some distros, use `python3.11` or `python3.12` explicitly.

## Install methods

=== "One-line installer"

    The fastest way to install Missy, clone the repo and install in one step:

    ```bash
    curl -fsSL https://raw.githubusercontent.com/MissyLabs/missy/master/install.sh | bash
    ```

    This clones the repository, creates a virtual environment, installs Missy, and launches the setup wizard.

=== "pip (editable)"

    Clone the repository and install in development/editable mode:

    ```bash
    git clone https://github.com/MissyLabs/missy.git
    cd missy
    pip install -e .
    ```

=== "pip (virtual environment)"

    Recommended for keeping your system Python clean:

    ```bash
    git clone https://github.com/MissyLabs/missy.git
    cd missy
    python3 -m venv .venv
    source .venv/bin/activate
    pip install -e .
    ```

!!! warning "Do not install as root"
    Missy stores configuration and data in `~/.missy/`. Installing as root will place these files under `/root/.missy/`, which is rarely what you want.

## Optional extras

Missy supports optional dependency groups for voice and observability features. Install them by appending the extra name in brackets:

=== "Voice support"

    Adds faster-whisper (STT), numpy, and soundfile for the voice channel:

    ```bash
    pip install -e ".[voice]"
    ```

    !!! note
        Voice support also requires the [Piper TTS](https://github.com/rhasspy/piper) binary, which is installed separately. See [Voice Server Setup](../channels/voice/server.md) for details.

=== "OpenTelemetry"

    Adds the OpenTelemetry SDK and OTLP exporters (gRPC and HTTP):

    ```bash
    pip install -e ".[otel]"
    ```

=== "Development"

    Adds pytest, ruff, mypy, and other development tools:

    ```bash
    pip install -e ".[dev]"
    ```

=== "Everything"

    Install all extras at once:

    ```bash
    pip install -e ".[dev,voice,otel]"
    ```

## Verify the installation

After installing, confirm that the `missy` CLI is available:

```bash
missy --help
```

You should see output listing all available commands. If the command is not found, ensure your virtual environment is activated or that the pip install location is on your `PATH`.

## Next steps

Run the setup wizard to configure your first AI provider:

```bash
missy setup
```

See [Quick Start](quickstart.md) for a walkthrough, or [Setup Wizard](setup-wizard.md) for the full reference.
