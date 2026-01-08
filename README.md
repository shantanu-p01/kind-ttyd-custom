# Custom KinD Image with ttyd

This directory contains files to create a custom KinD (Kubernetes in Docker) cluster with ttyd (terminal over HTTP) pre-installed.

**Docker Image:** https://hub.docker.com/r/shantanupatil01/custom-kind-ttyd

## Files

- **Dockerfile.custom-kind-ttyd**: Dockerfile to build custom KinD image with ttyd
- **kind-cluster-with-ttyd.yml**: KinD cluster configuration with ttyd port mapping
- **setup-kind-with-ttyd.sh**: Automated setup script

## Features

- ✅ KinD multi-node cluster (1 control-plane + 2 workers)
- ✅ ttyd installed and running on port **55555**
- ✅ Password authentication enabled
- ✅ NodePort mappings (30001-30010) for Kubernetes services
- ✅ Web-based terminal access

## Quick Start

### Option 1: Use Pre-built Image

1. **Pull the image:**
   ```bash
   docker pull shantanupatil01/custom-kind-ttyd:1.35.0
   ```

2. **Create the KinD cluster:**
   ```bash
   kind create cluster --config kind-cluster-with-ttyd.yml
   ```
   
   Note: The `kind-cluster-with-ttyd.yml` file already references the Docker Hub image

3. **Access ttyd:**
   - Open browser: http://localhost:55555
   - Username: `kind`
   - Password: `kind123`

### Option 2: Build from Source

1. **Build the custom image:**
   ```bash
   docker build -t custom-kind-ttyd:latest -f Dockerfile.custom-kind-ttyd .
   ```

2. **Create the KinD cluster:**
   ```bash
   kind create cluster --config kind-cluster-with-ttyd.yml
   ```

3. **Access ttyd:**
   - Open browser: http://localhost:55555
   - Username: `kind`
   - Password: `kind123`

## Configuration

### Changing the Password

Edit `Dockerfile.custom-kind-ttyd` and modify this line:
```dockerfile
RUN echo "kind:$(openssl passwd -apr1 YOUR_NEW_PASSWORD)" > /etc/ttyd.passwd
```

And update the ttyd command line:
```dockerfile
echo 'ttyd -p 55555 -c kind:YOUR_NEW_PASSWORD -t fontSize=16 bash > /var/log/ttyd.log 2>&1 &' >> /usr/local/bin/entrypoint-with-ttyd.sh && \
```

Then rebuild the image.

### Changing the Port

To change ttyd port from 55555 to another port:

1. Edit `Dockerfile.custom-kind-ttyd` - change `-p 55555` to your desired port
2. Edit `kind-cluster-with-ttyd.yml` - change containerPort and hostPort to match

## Default Credentials

- **Username:** kind
- **Password:** kind123

⚠️ **Important:** Change the default password for production use!

## Useful Commands

```bash
# View all clusters
kind get clusters

# Delete the cluster
kind delete cluster --name kind-cluster-ttyd

# Get cluster info
kubectl cluster-info --context kind-kind-cluster-ttyd

# List all nodes
kubectl get nodes

# Access a specific node
docker exec -it kind-cluster-ttyd-control-plane bash
```

## Port Mappings

| Service | Container Port | Host Port | Description |
|---------|---------------|-----------|-------------|
| ttyd | 55555 | 55555 | Web terminal |
| NodePort | 30001-30010 | 30001-30010 | Kubernetes services |

## Troubleshooting

### ttyd not accessible

Check if ttyd is running inside the container:
```bash
docker exec -it kind-cluster-ttyd-control-plane ps aux | grep ttyd
```

Check ttyd logs:
```bash
docker exec -it kind-cluster-ttyd-control-plane cat /var/log/ttyd.log
```

### Rebuild image after changes

```bash
# Delete existing cluster
kind delete cluster --name kind-cluster-ttyd

# Rebuild image
docker build -t custom-kind-ttyd:latest -f Dockerfile.custom-kind-ttyd .

# Create cluster again
kind create cluster --config kind-cluster-with-ttyd.yml
```

## Notes

- The ttyd service runs on all nodes (control-plane and workers)
- Only the control-plane node's ttyd is exposed on port 55555
- You can exec into worker nodes using `docker exec -it <container-name> bash`