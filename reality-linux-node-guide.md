# Running a Validator Node on Linux

In this guide, we will walk you through the process of setting up and running a validator node on the Reality Network using Linux. By following these steps, you'll be able to participate in the network and help secure the Reality blockchain.

## Prerequisites

Before you begin, ensure that you have:

- A Linux-based system (Ubuntu 22.04+ or Debian 12+ recommended)
- Minimum 8 GB RAM
- Minimum 160 GB disk space
- A stable internet connection
- Java 17 or higher installed

## VPS Recommendations

If you're running on a VPS, the following specifications are recommended:

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 vCores | 4 vCores |
| RAM | 8 GB | 16 GB |
| Storage | 160 GB SSD | 256 GB SSD |
| Network | 1 Gbps | 2.5 Gbps |
| Transfer | 5 TB/month | Unlimited |

Popular VPS providers that work well: Netcup, Hetzner, Contabo, DigitalOcean, Vultr.

---

## Step 1: Update Your System

First, update your system packages:

```bash
sudo apt update && sudo apt -y upgrade
```

Check if a reboot is required:

```bash
cat /var/run/reboot-required
```

If it shows "*** System restart required ***", reboot your server:

```bash
sudo reboot
```

---

## Step 2: Install Java

Reality Network requires Java 17 or higher. Install OpenJDK:

**For Ubuntu 22.04:**
```bash
sudo apt install openjdk-17-jre-headless -y
```

**For Debian 12/13 or Ubuntu 24.04:**
```bash
sudo apt install openjdk-21-jre-headless -y
```

Verify the installation:

```bash
java -version
```

You should see output showing Java 17+ or 21+.

---

## Step 3: Configure Your Firewall

Open the required ports for your validator node:

```bash
sudo apt install ufw -y
sudo ufw allow 22/tcp      # SSH (don't lock yourself out!)
sudo ufw allow 9000/tcp    # Public HTTP port
sudo ufw allow 9001/tcp    # P2P port
sudo ufw allow 9002/tcp    # CLI port
sudo ufw enable
sudo ufw status
```

---

## Step 4: Download the Reality Network JARs

Create a directory for your node and download the required files:

```bash
cd ~
mkdir reality-node && cd reality-node

# Download the core node software
wget https://github.com/reality-foundation/linux-server/releases/download/v0.13.0/reality-core-assembly-0.0.0+996-a9c70fe2.jar

# Download the keytool for generating your wallet
wget https://github.com/reality-foundation/linux-server/releases/download/v0.13.0/reality-keytool-assembly-0.0.0+996-a9c70fe2.jar

# Download the wallet tool for viewing your node ID
wget https://github.com/reality-foundation/linux-server/releases/download/v0.13.0/reality-wallet-assembly-0.0.0+996-a9c70fe2.jar
```

> **Note:** Check [GitHub Releases](https://github.com/reality-foundation/linux-server/releases) for the latest version.

---

## Step 5: Generate Your Keystore

Your keystore contains your node's private key and identity. Generate one using the keytool:

```bash
java -jar reality-keytool-assembly-0.0.0+996-a9c70fe2.jar generate \
  --keystore node.p12 \
  --keyalias node \
  --password YOUR_SECURE_PASSWORD
```

**Important:** 
- Replace `YOUR_SECURE_PASSWORD` with a strong, unique password
- Keep your password safe — you'll need it every time you start your node
- Back up your `node.p12` file securely — losing it means losing your node identity

---

## Step 6: Get Your Node ID

Your Node ID is your unique identifier on the network. Retrieve it using the wallet tool:

```bash
export CL_KEYSTORE=./node.p12
export CL_KEYALIAS=node
export CL_PASSWORD=YOUR_SECURE_PASSWORD

java -jar reality-wallet-assembly-0.0.0+996-a9c70fe2.jar show-id
```

This will output a long hexadecimal string — this is your Node ID. Save it somewhere; you'll need it to start your node.

Example output:
```
183d00412b31b5f3de360bf6545c99effa9d94dbdba48b2f36010ef264b5b1cb532d6a8816843999d7166ea4b4f70e834f6b50f4be3c5dc95479068b576fd720
```

---

## Step 7: Get Your Server's External IP

```bash
curl -4 -s ifconfig.me
```

> **Note:** The `-4` flag forces IPv4. If your server returns an IPv6 address (like `2a02:c207:...`), the node requires IPv4 instead.

Note this IP address — you'll need it for the next step.

---

## Step 8: Install tmux for Persistent Sessions

tmux allows your node to keep running after you disconnect from SSH:

```bash
sudo apt install tmux -y
```

---

## Step 9: Start Your Validator Node

You have two options for running your node:

### Option A: Systemd Service (Recommended for Production)

Systemd will auto-restart your node if it crashes and start it on boot.

First, create the service file:

```bash
sudo nano /etc/systemd/system/reality-node.service
```

Paste this (replace YOUR values):

```ini
[Unit]
Description=Reality Network Validator Node
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/reality-node
ExecStart=/usr/bin/java -Xms2g -Xmx2g -jar /root/reality-node/reality-core-assembly-0.0.0+996-a9c70fe2.jar run-validator --keystore /root/reality-node/node.p12 --password YOUR_PASSWORD --keyalias node --ip YOUR_EXTERNAL_IP --collateral 0 --peer-id YOUR_NODE_ID --l0-ip YOUR_EXTERNAL_IP --startup-port 9000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable reality-node
sudo systemctl start reality-node
```

Check status:

```bash
sudo systemctl status reality-node
```

View logs:

```bash
journalctl -u reality-node -f
```

### Option B: tmux (Quick Testing)

For quick testing, use tmux:

```bash
sudo apt install tmux -y
tmux new -s reality
```

Start your validator node (replace the placeholders with your actual values):

```bash
java -Xms2g -Xmx2g -jar reality-core-assembly-0.0.0+996-a9c70fe2.jar run-validator \
  --keystore node.p12 \
  --password YOUR_SECURE_PASSWORD \
  --keyalias node \
  --ip YOUR_EXTERNAL_IP \
  --collateral 0 \
  --peer-id YOUR_NODE_ID \
  --l0-ip YOUR_EXTERNAL_IP \
  --startup-port 9000
```

**Example with real values:**
```bash
java -Xms2g -Xmx2g -jar reality-core-assembly-0.0.0+996-a9c70fe2.jar run-validator \
  --keystore node.p12 \
  --password MySecurePass123 \
  --keyalias node \
  --ip 185.216.177.201 \
  --collateral 0 \
  --peer-id 183d00412b31b5f3de360bf6545c99effa9d94dbdba48b2f36010ef264b5b1cb532d6a8816843999d7166ea4b4f70e834f6b50f4be3c5dc95479068b576fd720 \
  --l0-ip 185.216.177.201 \
  --startup-port 9000
```

**Detach from tmux** (node keeps running in background):
Press `Ctrl+B`, then press `D`

**Reattach to tmux later:**
```bash
tmux attach -t reality
```

---

## Step 10: Verify Your Node is Running

Check your node's status by visiting:

```bash
curl http://YOUR_EXTERNAL_IP:9000/node/info
```

Or open in a browser: `http://YOUR_EXTERNAL_IP:9000/node/info`

You should see JSON output with your node information:
```json
{
  "state": "ReadyToJoin",
  "version": "0.0.0+996-a9c70fe2",
  "host": "185.216.177.201",
  "publicPort": 9000,
  "p2pPort": 9001,
  "id": "183d00412b31b5f3de360bf6545c99effa9d94dbdba48b2f36010ef264b5b1cb532d6a8816843999d7166ea4b4f70e834f6b50f4be3c5dc95479068b576fd720"
}
```

---

## Step 11: Join the Cluster

Once your node shows `"state": "ReadyToJoin"`, you can join the network cluster.

Run the join command (outside of tmux, in a separate terminal):

```bash
curl -X POST http://127.0.0.1:9002/cluster/join \
  -H 'Content-type: application/json' \
  -d '{
    "id": "0000003264c7c8503da3d03b6021101a57b5eb933d887bb7e3fbf4b2a57c302dfc5008afb522059b1926e8220de1cfa9388183de60b376a7bd93268990d71157",
    "ip": "143.110.227.9",
    "p2pPort": 9001
  }'
```

> **Note:** The peer ID and IP in this command are for the network's bootstrap node. Check the official documentation or community channels for current bootstrap node information.

---

## Step 12: Verify Cluster Join

Check your node status again:

```bash
curl http://YOUR_EXTERNAL_IP:9000/node/info
```

If successful, the state should change from `"ReadyToJoin"` to `"Observing"`:

```json
{
  "state": "Observing",
  "session": 1764683542634,
  "clusterSession": 1762520420862,
  ...
}
```

Your node is now part of the Reality Network! 🎉

---

## Node States

| State | Description |
|-------|-------------|
| `ReadyToJoin` | Node is running and ready to join the cluster |
| `Observing` | Node has joined and is syncing with the network |
| `Ready` | Node is fully synced and participating |

---

## Managing Your Node

### With Systemd (Recommended)

```bash
# Check status
sudo systemctl status reality-node

# View logs
journalctl -u reality-node -f

# Stop node
sudo systemctl stop reality-node

# Start node
sudo systemctl start reality-node

# Restart node
sudo systemctl restart reality-node
```

### With tmux

### Check Node Status
```bash
curl http://YOUR_EXTERNAL_IP:9000/node/info
```

### View Node Logs
```bash
tmux attach -t reality
```
(Press `Ctrl+B`, then `D` to detach again)

### Stop Your Node
```bash
tmux attach -t reality
# Press Ctrl+C to stop the node
```

### Restart Your Node
```bash
tmux attach -t reality
# Press Ctrl+C to stop, then run the java command again
```

---

## Creating a Startup Script (Optional)

For easier management, create a startup script:

```bash
nano ~/reality-node/start-node.sh
```

Add the following (replace with your values):

```bash
#!/bin/bash
cd ~/reality-node

export NODE_IP=$(curl -4 -s ifconfig.me)
export NODE_ID="YOUR_NODE_ID"
export NODE_PASSWORD="YOUR_SECURE_PASSWORD"

java -Xms2g -Xmx2g -jar reality-core-assembly-0.0.0+996-a9c70fe2.jar run-validator \
  --keystore node.p12 \
  --password $NODE_PASSWORD \
  --keyalias node \
  --ip $NODE_IP \
  --collateral 0 \
  --peer-id $NODE_ID \
  --l0-ip $NODE_IP \
  --startup-port 9000
```

Make it executable:

```bash
chmod +x ~/reality-node/start-node.sh
```

Now you can start your node with:

```bash
tmux new -s reality
~/reality-node/start-node.sh
```

---

## Troubleshooting

### Java not found
```bash
sudo apt install openjdk-21-jre-headless -y
```

### Port already in use
Check what's using the port:
```bash
sudo lsof -i :9000
```

### Node won't start
- Verify your keystore file exists: `ls -la node.p12`
- Check your password is correct
- Ensure all required ports are open: `sudo ufw status`

### Can't join cluster
- Verify your node is in `ReadyToJoin` state
- Check firewall ports are open
- Ensure you can reach the bootstrap node: `curl http://143.110.227.9:9000/node/info`

### Connection refused on port 9002
The CLI port (9002) only accepts connections from localhost (127.0.0.1). Make sure you're running the join command from the same server.

---

## Security Considerations

1. **Keep your keystore safe** — Back up `node.p12` securely
2. **Use strong passwords** — Your keystore password protects your node identity
3. **Keep your system updated** — Run `sudo apt update && sudo apt upgrade` regularly
4. **Firewall** — Only open necessary ports
5. **SSH hardening** — Consider using SSH keys instead of passwords

---

## Getting Help

- **Documentation:** [docs.realitynet.xyz](https://docs.realitynet.xyz)
- **GitHub:** [github.com/reality-foundation](https://github.com/reality-foundation)
- **Community:** Join the Reality Network community channels

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `java -jar reality-keytool-*.jar generate --keystore node.p12 --keyalias node --password PASS` | Generate keystore |
| `java -jar reality-wallet-*.jar show-id` | Show node ID (set env vars first) |
| `curl http://IP:9000/node/info` | Check node status |
| `curl -X POST http://127.0.0.1:9002/cluster/join ...` | Join cluster |
| `tmux new -s reality` | Start tmux session |
| `tmux attach -t reality` | Reattach to tmux |
| `Ctrl+B, D` | Detach from tmux |

---

*Last updated: December 2025*
*Guide version: 0.13.0*
