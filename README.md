# Netdata Monitoring Setup on AWS EC2

## Architecture

```text
Parent EC2 (10.0.8.78)
┌──────────────────────────────┐
│ Netdata Parent               │
│ Dashboard :19999             │
│ Receives streamed metrics    │
└──────────────▲───────────────┘
               │ Stream
               │
┌──────────────┴───────────────┐
│ Child EC2 (10.0.11.212)      │
│ Netdata Agent                │
│ Sends metrics to parent      │
└──────────────────────────────┘
```

## Prerequisites

- Ubuntu 22.04
- Two EC2 instances
- Same VPC
- Security Groups:
  - TCP 22 (SSH)
  - TCP 19999 between parent and child (or VPC CIDR)
- Outbound: All traffic

---

# 1. Install Netdata (Run on BOTH Parent and Child)

```bash
sudo apt update
sudo apt upgrade -y

wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh
chmod +x /tmp/netdata-kickstart.sh
sudo /tmp/netdata-kickstart.sh

sudo systemctl enable netdata
sudo systemctl restart netdata
sudo systemctl status netdata
```

After installation, log in to the Netdata UI and claim the node using the following session key:

```bash
sudo cat /var/lib/netdata/netdata_random_session_id
```

Use the displayed key when prompted to claim the node in the Netdata dashboard.

Verify:

```bash
sudo ss -tulpn | grep 19999
```

Open:

```
http://<PUBLIC-IP>:19999
```

---

# 2. Configure Parent

Generate API key:

```bash
uuidgen
```

Example:

```
2c5b52d0-fc18-4b2d-99c3-9f82d67f4f31
```

Edit:

```bash
sudo nano /etc/netdata/stream.conf
```

```ini
[2c5b52d0-fc18-4b2d-99c3-9f82d67f4f31]
    enabled = yes
    allow from = 10.0.0.0/16
    default history = 3600
```

Restart:

```bash
sudo systemctl restart netdata
```

---

# 3. Configure Child

```bash
sudo nano /etc/netdata/stream.conf
```

```ini
[stream]
    enabled = yes
    destination = 10.0.8.78:19999
    api key = 2c5b52d0-fc18-4b2d-99c3-9f82d67f4f31
```

Restart:

```bash
sudo systemctl restart netdata
```

---

# 4. Verify Streaming

Parent:

```bash
sudo journalctl -u netdata -f
```

Expected:

```
STREAM RCV 'ip-10-0-11-212'
```

Child:

```bash
sudo journalctl -u netdata -f
```

No repeated stream errors should appear.

---

# 5. Verify Dashboard

Parent:
- CPU
- Memory
- Disk
- Network
- Processes

Generate test load:

```bash
sudo apt install stress -y
stress --cpu 2 --timeout 60
stress --vm 2 --vm-bytes 512M --timeout 60
```

Disk:

```bash
dd if=/dev/zero of=test.img bs=1M count=512
rm test.img
```

---

# Useful Commands

Status

```bash
sudo systemctl status netdata
```

Restart

```bash
sudo systemctl restart netdata
```

Logs

```bash
sudo journalctl -u netdata -f
```

Version

```bash
netdata -V
```

Port

```bash
sudo ss -tulpn | grep 19999
```

---

# Metrics Observed

- CPU Utilization
- Memory Usage
- Disk Read/Write
- Network Throughput
- Processes
- System Load
- Uptime

---

# Conclusion

Successfully deployed Netdata on two AWS EC2 Ubuntu instances. The parent node receives streamed metrics from the child node and provides centralized real-time monitoring through the Netdata dashboard.
