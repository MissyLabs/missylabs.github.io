---
tags:
  - edge
  - setup
---

# Pi Setup

## Automated install

The `setup_pi.sh` script handles everything:

```bash
git clone https://github.com/MissyLabs/missy-edge.git
cd missy-edge
sudo bash setup_pi.sh
```

The script:

1. Creates a `missy-edge` system user with no login shell.
2. Installs system dependencies: `portaudio19-dev`, `libasound2-dev`, `libsndfile1`.
3. Creates `/etc/missy-edge/` with restrictive permissions (`0700`).
4. Sets up a Python venv at `/opt/missy-edge/venv/`.
5. Installs missy-edge and its dependencies into the venv.
6. Installs and enables the systemd service.

## Manual install

If you prefer to set things up yourself:

### System dependencies

```bash
sudo apt update
sudo apt install -y python3-venv python3-dev portaudio19-dev \
    libasound2-dev libsndfile1 git
```

### Create the venv

```bash
sudo mkdir -p /opt/missy-edge
sudo python3 -m venv /opt/missy-edge/venv
sudo /opt/missy-edge/venv/bin/pip install -e ".[wakeword]"
```

### Create config directory

```bash
sudo mkdir -p /etc/missy-edge
sudo chmod 700 /etc/missy-edge
```

### Create the config file

```bash
sudo tee /etc/missy-edge/config.yaml > /dev/null <<'EOF'
server_url: "ws://YOUR_MISSY_SERVER:8765"
name: "Living Room Pi"
room: "Living Room"
EOF
sudo chmod 600 /etc/missy-edge/config.yaml
```

## systemd service

The setup script installs a service unit at `/etc/systemd/system/missy-edge.service`:

```ini
[Unit]
Description=Missy Edge Voice Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=missy-edge
ExecStart=/opt/missy-edge/venv/bin/missy-edge
Restart=always
RestartSec=5

# Security hardening
ProtectSystem=strict
ProtectHome=true
NoNewPrivileges=true
ReadOnlyPaths=/
ReadWritePaths=/etc/missy-edge

[Install]
WantedBy=multi-user.target
```

### Service commands

```bash
# Start the service
sudo systemctl start missy-edge

# Stop the service
sudo systemctl stop missy-edge

# View logs
sudo journalctl -u missy-edge -f

# Enable on boot
sudo systemctl enable missy-edge

# Check status
sudo systemctl status missy-edge
```

## Development setup

For local development and testing (not on a Pi):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# Run tests (56 tests, no hardware required)
python -m pytest tests/ -v
```

### Running locally

```bash
# Wake word mode (requires microphone + wake word model)
missy-edge

# Push-to-talk mode (press Enter to speak)
missy-edge --manual

# Override server URL
missy-edge --server ws://localhost:8765

# Verbose logging
missy-edge -v
```

## Verifying the installation

After setup, verify the key files are in place:

```bash
# Config directory exists with correct permissions
ls -la /etc/missy-edge/
# Should show: drwx------ root root

# Venv is functional
/opt/missy-edge/venv/bin/missy-edge --help

# Service is recognized
sudo systemctl status missy-edge
```

## Next steps

After the Pi is set up, [pair it with your Missy server](pairing.md).
