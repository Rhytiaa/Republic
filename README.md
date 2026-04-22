# Republic AI Testnet Node Installation Guide (`raitestnet_77701-1`)

## 📋 Hardware Requirements

| Component | Minimum | Recommended |
| :--- | :--- | :--- |
| **CPU** | 4 Cores | 8 Cores |
| **RAM** | 8 GB | 16 GB |
| **SSD** | 200 GB NVMe | 500 GB NVMe |
| **OS** | Ubuntu 24.04 | Ubuntu 24.04 |

## 🛠 Installation Steps

### 1\. System Update & Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl build-essential make jq gcc snapd chrony lz4 tmux unzip bc -y
```

### 2\. Install Go

```bash
rm -rf $HOME/go
sudo rm -rf /usr/local/go
cd $HOME
curl https://dl.google.com/go/go1.24.5.linux-amd64.tar.gz | sudo tar -C /usr/local -zxvf -

# Setup environment
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
```

### 3\. Install Binary & Cosmovisor

We use **Cosmovisor** to ensure the node handles future upgrades automatically.

```bash
# Install Cosmovisor
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest

# Download Republic Binary
cd $HOME
wget https://ss-t.republicai.nodestake.org/republicd
chmod +x republicd

# Prepare Cosmovisor structure
mkdir -p $HOME/.republic/cosmovisor/genesis/bin
mkdir -p $HOME/.republic/cosmovisor/upgrades
mv republicd $HOME/.republic/cosmovisor/genesis/bin/

# Create symlink for easy access
sudo ln -s $HOME/.republic/cosmovisor/genesis/bin/republicd /usr/local/bin/republicd
```

### 4\. Initialize Node

Replace `NodeName` with your own moniker.

```bash
republicd init NodeName --chain-id=raitestnet_77701-1

# Download Genesis and Addrbook
curl -Ls https://ss-t.republicai.nodestake.org/genesis.json > $HOME/.republic/config/genesis.json 
curl -Ls https://ss-t.republicai.nodestake.org/addrbook.json > $HOME/.republic/config/addrbook.json
```

### 5\. Create Systemd Service (Cosmovisor)

```bash
sudo tee /etc/systemd/system/republicd.service > /dev/null <<EOF
[Unit]
Description=Republic AI Node (Cosmovisor)
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/.republic
ExecStart=$(which cosmovisor) run start
Restart=always
RestartSec=3
LimitNOFILE=65535
Environment="DAEMON_NAME=republicd"
Environment="DAEMON_HOME=$HOME/.republic"
Environment="DAEMON_ALLOW_STRUCTURAL_ADDITIONS=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable republicd
```

### 6\. Download Snapshot (Optional)

```bash
SNAP_NAME=$(curl -s https://ss-t.republicai.nodestake.org/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss-t.republicai.nodestake.org/${SNAP_NAME} | lz4 -c -d - | tar -x -C $HOME/.republic
```

### 7\. Launch Node

```bash
sudo systemctl restart republicd
journalctl -u republicd -f
```

-----

## 📊 Useful Commands

### Check Sync Status

```bash
republicd status 2>&1 | jq .SyncInfo
```

### Wallet Management

  * **Create New:** `republicd keys add wallet`
  * **Restore:** `republicd keys add wallet --recover`

### Service Controls

  * **Restart:** `sudo systemctl restart republicd`
  * **Stop:** `sudo systemctl stop republicd`
  * **Logs:** `journalctl -u republicd -f`

-----

## 🗑 Uninstall Node

```bash
sudo systemctl stop republicd
sudo systemctl disable republicd
sudo rm /etc/systemd/system/republicd.service
sudo rm /usr/local/bin/republicd
rm -rf $HOME/.republic
rm -rf $HOME/go/bin/republicd
```
