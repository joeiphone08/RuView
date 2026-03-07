# Raspberry Pi 5 Setup Guide

Run WiFi-DensePose on a Raspberry Pi 5 (16GB) using built-in WiFi for real-time people detection via RSSI sensing.

---

## Requirements

| Component | Specification |
|-----------|--------------|
| **Board** | Raspberry Pi 5, 16GB RAM |
| **OS** | Raspberry Pi OS (64-bit, Bookworm) |
| **Storage** | 16GB+ microSD or NVMe SSD |
| **Network** | WiFi enabled (built-in 802.11ac) |
| **Cooling** | Active fan recommended for 24/7 operation |

No ESP32 or external hardware required. The Pi's built-in WiFi handles RSSI-based sensing.

---

## What It Detects

| Capability | Description |
|-----------|-------------|
| **Presence** | Is someone in the room (yes/no) |
| **Motion** | Still, moving, or walking |
| **Breathing** | Coarse respiratory rate (stationary subject near AP) |

Detection works by scanning nearby WiFi access points and analyzing signal strength changes caused by human movement.

---

## Installation

### Step 1: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### Step 2: Log out and back in

```bash
logout
```

This is required for the Docker group permission to take effect.

### Step 3: Run the sensing server

```bash
docker run -d --network host --name ruview ruvnet/wifi-densepose:latest --source linux --tick-ms 500
```

| Flag | Purpose |
|------|---------|
| `-d` | Run in background |
| `--network host` | Access Pi's WiFi interface directly |
| `--name ruview` | Container name for easy management |
| `--source linux` | Use Pi's built-in WiFi for real RSSI sensing |
| `--tick-ms 500` | Scan every 500ms (2 Hz) |

### Step 4: Verify it's running

```bash
docker logs ruview
curl http://localhost:3000/health
```

### Step 5: Open the UI

From any device on your network, open a browser and go to:

```
http://<your-pi-ip>:3000
```

To find your Pi's IP address:

```bash
hostname -I
```

---

## API Endpoints

```bash
# Health check
curl http://localhost:3000/health

# Latest sensing frame
curl http://localhost:3000/api/v1/sensing/latest

# Vital signs (breathing rate)
curl http://localhost:3000/api/v1/vital-signs

# Pose data (17 COCO keypoints)
curl http://localhost:3000/api/v1/pose/current

# Multi-BSSID registry (visible access points)
curl http://localhost:3000/api/v1/bssid

# Server info
curl http://localhost:3000/api/v1/info
```

All endpoints return JSON.

---

## Management Commands

```bash
# Stop the service
docker stop ruview

# Start the service
docker start ruview

# View live logs
docker logs -f ruview

# Restart the service
docker restart ruview

# Remove and recreate
docker rm -f ruview
docker run -d --network host --name ruview ruvnet/wifi-densepose:latest --source linux --tick-ms 500

# Update to latest version
docker pull ruvnet/wifi-densepose:latest
docker rm -f ruview
docker run -d --network host --name ruview ruvnet/wifi-densepose:latest --source linux --tick-ms 500
```

---

## Auto-Start on Boot

To have the service start automatically when the Pi boots:

```bash
docker update --restart unless-stopped ruview
```

To disable auto-start:

```bash
docker update --restart no ruview
```

---

## Troubleshooting

### Container exits immediately

Check logs for errors:

```bash
docker logs ruview
```

WiFi scanning requires network access. Ensure `--network host` is set.

### No access points detected

The Linux WiFi scanner uses `iw dev wlan0 scan`, which requires `CAP_NET_ADMIN`. The `--network host` flag should provide this inside Docker. If not:

```bash
docker rm -f ruview
docker run -d --network host --privileged --name ruview ruvnet/wifi-densepose:latest --source linux --tick-ms 500
```

### WiFi interface not found

Check your interface name:

```bash
iw dev
```

If it's not `wlan0` (e.g., `wlp1s0`), you may need to configure the interface name in the container.

### Cannot reach the UI from another device

Ensure the Pi and your device are on the same network. Check the Pi's IP:

```bash
hostname -I
```

Check that port 3000 is not blocked:

```bash
sudo ss -tlnp | grep 3000
```

### Thermal throttling under load

Check CPU temperature:

```bash
vcgencmd measure_temp
```

If above 80C, ensure active cooling is installed. The Pi 5 active cooler or any compatible fan will prevent throttling.

---

## Performance Notes

- **RAM usage**: ~100-200MB (16GB is more than enough)
- **CPU**: Light load at 2 Hz scan rate; well within Pi 5 Cortex-A76 capability
- **Scan rate**: `--tick-ms 500` = 2 scans/second. Lower values (e.g., `250`) increase sensitivity but use more CPU
- **Best results**: 3+ visible WiFi access points in range for spatial correlation
- **Optimal placement**: Central location with line of sight to monitored area
