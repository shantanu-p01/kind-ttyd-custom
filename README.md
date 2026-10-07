# Custom KinD Image with ttyd and SSH

This directory contains files to create a custom KinD (Kubernetes in Docker) cluster with ttyd (terminal over HTTP) and passwordless SSH between nodes.

**Docker Image:** https://hub.docker.com/r/shantanupatil01/custom-kind-ttyd

## Files

- **Dockerfile.custom-kind-ttyd**: Dockerfile to build custom KinD image with ttyd and SSH
- **kind-cluster-with-ttyd.yml**: KinD cluster configuration with ttyd port mapping

## Features

- ✅ KinD multi-node cluster (1 control-plane + 2 workers)
- ✅ ttyd installed and running on port **55555** (control-plane only)
- ✅ **Passwordless SSH** between all nodes (control-plane ↔ workers)
- ✅ **Automatic node discovery** (updates /etc/hosts every 500ms)
- ✅ **Clean hostname prompts** (control-plane, worker01, worker02)
- ✅ **Custom Kubernetes node names** (control-plane, worker01, worker02)
- ✅ **kubectl alias `k`** with bash autocomplete (control-plane only)
- ✅ Password authentication enabled
- ✅ NodePort mappings (30001-30025) for Kubernetes services
- ✅ Web-based terminal access
- ✅ HOME directory properly set to /root

## Quick Start

### Option 1: Use Pre-built Image

1. **Create the KinD cluster:**
   ```bash
   kind create cluster --config kind-cluster-with-ttyd.yml
   ```
   
   Note: The config automatically pulls `shantanupatil01/custom-kind-ttyd:1.37.0` from your local build

2. **Access control-plane via ttyd:**
   - Open browser: http://localhost:55555
   - Username: `kind`
   - Password: `kind123`

3. **SSH between nodes (from control-plane):**
   ```bash
   ssh worker01    # SSH to first worker
   ssh worker02    # SSH to second worker
   ssh control-plane  # SSH back to control-plane
   ```
   No password required - uses pre-generated SSH keys!

### Option 2: Build from Source

1. **Build the custom image locally:**
   ```bash
   docker buildx build --platform linux/amd64 -t shantanupatil01/custom-kind-ttyd:1.37.0 -f Dockerfile.custom-kind-ttyd --load .
   ```

2. **Create the KinD cluster:**
   ```bash
   kind create cluster --config kind-cluster-with-ttyd.yml
   ```

3. **Access via ttyd and SSH as above**

## SSH Configuration

### Automatic Node Discovery

The image includes a systemd service that:
- Runs every **500ms** to discover cluster nodes
- Automatically updates `/etc/hosts` with short names:
  - `control-plane` → control-plane node
  - `worker01` → first worker
  - `worker02` → second worker
- Starts SSH server automatically on all nodes

### Passwordless SSH

All nodes have:
- Pre-generated SSH keypair embedded in the image
- Public key added to `authorized_keys`
- No password required for inter-node SSH

### Clean Shell Prompts

Terminals show color-coded short hostnames:
```bash
control-plane:~$
worker01:~$
worker02:~$
```

## Configuration

### Changing Credentials & Password Management

You have several ways to set or change credentials without needing to rebuild the image:

#### Method 1: Mandatory Credential Setup on First Login (Enforced)
Upon your very first interactive login via the web terminal or SSH, credential setup is strictly enforced and cannot be bypassed:
```text
┌─────────────────────────────────────────────────────────────────┐
│ ⚠️  SECURITY NOTICE: Mandatory Credential Setup                 │
│ Default credentials (kind / kind123) are currently active.       │
│ You must configure a new username and password to proceed.       │
└─────────────────────────────────────────────────────────────────┘

Enter new username [current: kind]: 
Enter new password: 
Confirm new password: 
```
- You can specify a custom username or press Enter to keep `kind`.
- You must enter and confirm a new password (`kind123` is disallowed).
- Verification cannot be cancelled or bypassed (Ctrl+C and Ctrl+D are handled).
- Once configured, the script updates the credentials, restarts `ttyd`, and reloads the web session.

#### Method 2: Change Credentials Anytime via Terminal CLI
Run the built-in utility inside the container at any time:
```bash
# Interactive mode (prompts for username and password)
change-password

# Non-interactive mode (specify username and/or password)
change-password -u admin -p "MyNewSecretPassword"
change-password -p "MyNewSecretPassword"
```

#### Method 3: Change from Host via Docker CLI
Change credentials from your host machine without opening the web terminal:
```bash
docker exec kind-cluster-ttyd-control-plane change-password -u admin -p "MyNewSecretPassword"
```

#### Method 4: Build-time Arguments (If Building from Source)
Override default credentials during `docker buildx` without editing the Dockerfile:
```bash
docker buildx build \
  --build-arg DEFAULT_USER=admin \
  --build-arg DEFAULT_PASSWORD=custompassword \
  ...
```

### Changing Discovery Interval

To change node discovery interval from 500ms:

Edit the systemd service in `Dockerfile.custom-kind-ttyd`:
```dockerfile
echo 'ExecStart=/bin/bash -c "while true; do /usr/local/bin/kind-node-discovery.sh; sleep 0.5; done"' 
# Change 0.5 to your desired interval in seconds
```

### Adding More Workers

Edit `kind-cluster-with-ttyd.yml` and add more worker nodes:
```yaml
  - role: worker
    image: shantanupatil01/custom-kind-ttyd:1.37.0
    extraPortMappings:
      - containerPort: 30031
        hostPort: 30031
        listenAddress: "0.0.0.0"
        protocol: tcp
```

The discovery service will automatically detect up to 5 workers (worker01-worker05).

## Default Credentials

- **ttyd Username:** kind
- **ttyd Password:** kind123
- **SSH:** Passwordless (key-based authentication)

⚠️ **Important:** Change the default password for production use!

## Useful Commands

```bash
# View all clusters
kind get clusters

# Delete the cluster
kind delete cluster --name kind-cluster-ttyd

# Get cluster info
kubectl cluster-info --context kind-kind-cluster-ttyd

# List all nodes (shows custom names: control-plane, worker01, worker02)
kubectl get nodes
# or use the alias from control-plane:
k get nodes

# Access control-plane directly
docker exec -it kind-cluster-ttyd-control-plane bash

# Check node discovery service status
docker exec kind-cluster-ttyd-control-plane systemctl status kind-node-discovery.service

# View /etc/hosts entries on control-plane
docker exec kind-cluster-ttyd-control-plane cat /etc/hosts
```

## Port Mappings

| Service | Container Port | Host Port | Description |
|---------|---------------|-----------|-------------|
| ttyd (control-plane) | 55555 | 55555 | Web terminal |
| NodePort (control-plane) | 30001-30010 | 30001-30010 | Kubernetes services |
| NodePort (worker-1) | 30011-30015 | 30011-30015 | Kubernetes services |
| NodePort (worker-2) | 30021-30025 | 30021-30025 | Kubernetes services |

**Note:** SSH ports (22) are NOT exposed on the host - only accessible between containers internally.

## Troubleshooting

### ttyd not accessible

Check if ttyd is running inside the control-plane:
```bash
docker exec kind-cluster-ttyd-control-plane systemctl status ttyd.service
```

### SSH not working between nodes

Check if sshd is running:
```bash
docker exec kind-cluster-ttyd-control-plane pgrep sshd
docker exec kind-cluster-ttyd-worker pgrep sshd
```

Check node discovery:
```bash
docker exec kind-cluster-ttyd-control-plane cat /etc/hosts | grep kind-ssh-shortcuts
```

Manually restart discovery service:
```bash
docker exec kind-cluster-ttyd-control-plane systemctl restart kind-node-discovery.service
```

### HOME not set error

This should be fixed in the latest image. If you see this error, rebuild:
```bash
docker buildx build --platform linux/amd64 -t shantanupatil01/custom-kind-ttyd:1.37.0 -f Dockerfile.custom-kind-ttyd --load .
kind delete cluster --name kind-cluster-ttyd
kind create cluster --config kind-cluster-with-ttyd.yml
```

## Architecture

```
┌─────────────────────────────────────────┐
│     Host Machine (localhost:55555)      │
└────────────────┬────────────────────────┘
                 │ ttyd web access
                 ▼
    ┌────────────────────────────┐
    │   control-plane            │
    │   - ttyd:55555             │◄──SSH──┐
    │   - sshd:22                │        │
    │   - node-discovery service │        │
    └────────────┬───────────────┘        │
                 │                        │
       ┌─────────┴──────────┐             │
       │  Passwordless SSH  │             │
       ▼                    ▼             │
┌──────────────┐    ┌──────────────┐      │
│  worker01    │    │  worker02    │      │
│  - sshd:22   │◄───┤  - sshd:22   │──────┘
│  - discovery │    │  - discovery │
└──────────────┘    └──────────────┘
```

## Notes

- The ttyd service runs only on the control-plane node
- SSH is enabled on all nodes with passwordless authentication
- Node discovery runs on all nodes and updates /etc/hosts automatically
- All nodes share the same SSH keypair (embedded in the image at build time)
- Default working directory is `/root` for all nodes
- KUBECONFIG is pre-configured for kubectl access

## Contact

For questions, issues, or suggestions:
- **Email:** shantanu.verulkar.01@gmail.com