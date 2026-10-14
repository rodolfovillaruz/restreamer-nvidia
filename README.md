# Restreamer on Kubernetes with NVIDIA GPU

## Prerequisites (once per node — run on the GPU node, e.g. `rog`)

1. The NVIDIA driver and `nvidia-container-toolkit` are installed, and `nvidia-smi`
   works on the node (reboot after a driver upgrade).

2. Register the `nvidia` runtime with containerd (skip if
   `sudo containerd config dump | grep nvidia` already shows it):

   ```sh
   sudo nvidia-ctk runtime configure --runtime=containerd
   sudo systemctl restart containerd
   ```

3. Install the NVIDIA device plugin so the node advertises `nvidia.com/gpu`:

   ```sh
   kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.1/deployments/static/nvidia-device-plugin.yml
   kubectl patch ds -n kube-system nvidia-device-plugin-daemonset \
     --type merge -p '{"spec":{"template":{"spec":{"runtimeClassName":"nvidia"}}}}'
   ```

   The patch needs the `RuntimeClass` from `restreamer.yaml`, so apply that first.
   Check with:

   ```sh
   kubectl get node -o jsonpath='{.items[*].status.allocatable.nvidia\.com/gpu}'   # → 1
   ```

## Deploy

```sh
kubectl apply -f restreamer.yaml
kubectl -n restreamer rollout status deploy/restreamer
kubectl -n restreamer exec deploy/restreamer -- nvidia-smi
```

## Access (NodePort)

| Service | Container | Node port |
|---------|-----------|-----------|
| UI/API (HTTP)  | 8080     | 30080 |
| UI/API (HTTPS) | 8181     | 30181 |
| RTMP           | 1935     | 31935 |
| RTMPS          | 1936     | 31936 |
| SRT            | 6000/udp | 30600 |

UI: `http://<node-ip>:30080/ui`. OBS: `rtmp://<node-ip>:31935/...`

State is stored on the node in `/var/lib/restreamer/{config,data}`.
