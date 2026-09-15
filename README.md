🌐 Worrell Testnet Node & Validator Setup Guide

This repository contains a step-by-step guide for deploying a full node, performing a fast State Sync, and creating a validator on the Worrell Testnet.

📊 Network Details

Chain ID: worrell-testnet-1

Binary Name: worrelld (Cosmos SDK v0.53.6)

Base Denom: uworrell (1 WORRELL = 1,000,000 uworrell)

Min Gas Price: 0.025uworrell

💻 System Requirements

Resource: Testnet (Recommended)
OS: Ubuntu 22.04 LTS+

CPU: 2 vCPU

RAM: 4 GB

Storage: 100 GB SSD

🛠️ 1. Preparation & Installation
Update your server and install the necessary dependencies:

```Bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git jq lz4 build-essential -y
```
Download and install the prebuilt binary:

```Bash
curl -LO https://github.com/worrellchain/worrell/releases/download/v0.1.2/worrell_v0.1.2_linux_amd64.tar.gz
tar -xzf worrell_v0.1.2_linux_amd64.tar.gz
sudo mv worrelld /usr/local/bin/

# Check version (cosmos_sdk_version should be v0.53.6)
worrelld version --long | head -5
```

⚙️ 2. Initialization & Configuration
Set your preferred MONIKER (Node Name) and WALLET (Wallet Name) by configuring these environment variables:

```Bash
export MONIKER="YOUR_NODE_NAME"
export WALLET="YOUR_WALLET_NAME"
```
Initialize the node and download the official Genesis file:

```Bash
# Initialize Node
worrelld init $MONIKER --chain-id worrell-testnet-1

# Download official Genesis file
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-1/genesis.json > $HOME/.worrell/config/genesis.json

# Verify Genesis Hash (Output should start with a81c...)
sha256sum $HOME/.worrell/config/genesis.json
```
Configure Persistent Peers and Minimum Gas Price (Includes official and community peers):

```Bash
# Set Persistent Peers
PEERS="bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656,40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656,e812f08760b18ed774369e899763735f80179f76@peer-worrell.grandvalleys.com:17656"
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"$PEERS\"|" $HOME/.worrell/config/config.toml

# Set Minimum Gas Price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025uworrell\"|" $HOME/.worrell/config/app.toml
```

⚡ 3. Fast Syncing via State Sync
Syncing from block zero can take hours. We will use State Sync via the ITRocket RPC to join the network quickly:

```Bash
# Reset node data (keeps priv_validator_key intact)
worrelld tendermint unsafe-reset-all --home $HOME/.worrell

SNAP_RPC="https://worrell-testnet-rpc.itrocket.net:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
TRUST_HEIGHT=$((LATEST_HEIGHT - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)

sed -i -E "s|^[[:space:]]*enable[[:space:]]*=.*|enable = true|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*rpc_servers[[:space:]]*=.*|rpc_servers = \"$SNAP_RPC,$SNAP_RPC\"|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*trust_height[[:space:]]*=.*|trust_height = $TRUST_HEIGHT|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*trust_hash[[:space:]]*=.*|trust_hash = \"$TRUST_HASH\"|" $HOME/.worrell/config/config.toml
```

🛡️ 4. Systemd Service Setup
Create a systemd service file to run the node in the background continuously:
```Bash
sudo tee /etc/systemd/system/worrelld.service > /dev/null <<EOF
[Unit]
Description=Worrell Testnet Node
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/worrelld start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable --now worrelld
```

💰 5. Wallet & Validator Creation
Wallet Operations (Create or Import)

Option A: Create a New Wallet

```Bash
# MAKE SURE TO BACKUP THE 24-WORD SEED PHRASE GENERATED HERE!
worrelld keys add $WALLET
```

Option B: Import an Existing Wallet (Recover)

```Bash
# After running this command, you will be prompted to enter your 24-word seed phrase.
worrelld keys add $WALLET --recover
```

Faucet & Balance Check:

```Bash
# Request test tokens from the Faucet (500 WORRELL available per hour)
curl -X POST http://164.68.98.186:4500 \
  -H "Content-Type: application/json" \
  -d "{\"address\":\"$(worrelld keys show $WALLET -a)\"}"

# Check your balance
worrelld query bank balances $(worrelld keys show $WALLET -a)
```

IMPORTANT: Ensure your node is fully synced before creating a validator. The catching_up status must be false. (See the Monitoring section below).

Create Validator
Once your node is synced and your wallet has funds (e.g., using 490 WORRELL):

```Bash
cat <<EOF > $HOME/.worrell/validator.json
{
  "pubkey": $(worrelld tendermint show-validator),
  "amount": "490000000uworrell",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "Worrell Node Operator",
  "commission-rate": "0.05",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1000000"
}
EOF

worrelld tx staking create-validator $HOME/.worrell/validator.json \
  --from $WALLET \
  --chain-id worrell-testnet-1 \
  --gas auto \
  --gas-adjustment 1.5 \
  --gas-prices 0.025uworrell \
  -y
```

🔍 6. Monitoring & Operations Cheatsheet
Node Monitoring & Logs

```Bash
# View live logs
sudo journalctl -u worrelld -f -o cat

# Check sync status ("catching_up": false means you are fully synced)
worrelld status 2>&1 | jq '.sync_info'

# Check validator signing info and missed blocks
worrelld query slashing signing-info $(worrelld tendermint show-address)
```

Validator Management

```Bash
# Delegate more tokens to your validator (e.g., 100 WORRELL)
worrelld tx staking delegate $(worrelld keys show $WALLET --bech val -a) 100000000uworrell \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y

# Unjail your validator
worrelld tx slashing unjail \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y

# Update commission rate (Max 1% change per day. e.g., to 6%)
worrelld tx staking edit-validator \
  --commission-rate 0.06 \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y
```

🔄 7. Updates and Deletion

A. Updating the Binary (When a new release is announced)
If the network upgrades (e.g., to v0.2.0), stop the node and replace the binary:

```Bash
sudo systemctl stop worrelld
# Download the new release and overwrite the old binary
curl -LO https://github.com/worrellchain/worrell/releases/download/vX.Y.Z/worrell_vX.Y.Z_linux_amd64.tar.gz
tar -xzf worrell_vX.Y.Z_linux_amd64.tar.gz
sudo mv worrelld /usr/local/bin/
# Restart the service
sudo systemctl restart worrelld
sudo journalctl -u worrelld -f -o cat
```

B. Completely Remove the Node (Use with Caution!)
If you want to wipe the node entirely from your server (Ensure you have your seed phrase backed up!):

```Bash
sudo systemctl stop worrelld
sudo systemctl disable worrelld
sudo rm /etc/systemd/system/worrelld.service
sudo systemctl daemon-reload
rm -rf $HOME/.worrell
sudo rm /usr/local/bin/worrelld
```
