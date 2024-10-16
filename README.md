# Nolus Validator Node Setup Guide
```markdown
# Nolus Validator Node Setup Guide

This README provides detailed instructions on setting up a Nolus validator node.

## Table of Contents
- [Hardware Requirements](#hardware-requirements)
- [Prerequisites](#prerequisites)
- [Build Nolus Core](#build-nolus-core)
- [Configuration and Execution](#configuration-and-execution)
- [Set Up Cosmovisor (Optional)](#set-up-cosmovisor-optional)
- [Run Your Node](#run-your-node)
- [Run Your Node From a Snapshot (Optional)](#run-your-node-from-a-snapshot-optional)
- [Create a Validator](#create-a-validator)
- [Secure Your Keys](#secure-your-keys)
- [Restore a Validator](#restore-a-validator)
- [Unjail a Validator](#unjail-a-validator)

## Hardware Requirements

To run a Nolus node, the following minimum hardware specifications are recommended:

- **CPU**: 4+ vCPU
- **RAM**: 16+ GB
- **Storage**: 240+ GB SSD

## Prerequisites

Ensure you have the following installed:

- **Golang**: Version 1.21.5 for linux/amd64

### Install Build Tools (Linux Users)

```bash
sudo apt-get install -y build-essential
```

## Build Nolus Core

1. **Clone the Nolus Core Repository**

   ```bash
   git clone https://github.com/Nolus-Protocol/nolus-core
   ```

2. **Navigate to the Core Directory**

   ```bash
   cd nolus-core
   ```

3. **Check Out the Latest Stable Release**

   Replace `[latest version]` with the desired version (e.g., `v0.6.2` or `v0.3.0`).

   ```bash
   git checkout [latest version]
   ```

4. **Compile and Install Nolus Core**

   ```bash
   make install
   ```

5. **Verify Installation**

   ```bash
   nolusd version --long
   ```

   If you see an error indicating `nolusd: command not found`, set the Go binary path:

   ```bash
   export PATH=$PATH:$(go env GOPATH)/bin
   ```

## Configuration and Execution

1. **Initialize a Nolus Node**

   ```bash
   nolusd init "My First Nolus Node"
   ```

   This creates configuration files in the `.nolus` directory.

2. **Obtain the Genesis File**

   Download the genesis state file:

   ```bash
   wget https://raw.githubusercontent.com/nolus-protocol/nolus-networks/main/mainnet/pirin-1/genesis.json
   ```

   Move it to the appropriate directory:

   ```bash
   mv ./genesis.json ~/.nolus/config/genesis.json
   ```

   You can also download the `addrbook.json` file for peer synchronization:

   ```bash
   wget https://github.com/nolus-protocol/nolus-networks/blob/main/mainnet/pirin-1/addrbook.json
   ```

3. **Set Persistent Peers**

   Use the provided peer list:

   ```bash
   PEERS="89757803f40da51678451735445ad40d5b15e059@169.155.168.149:26656,c6be81e1757c31012ef201d396981d69d370f37a@162.19.237.150:26656,..."
   ```

   Update the configuration file:

   ```bash
   sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"$PEERS\"|" $HOME/.nolus/config/config.toml
   ```

4. **Edit Configuration Files**

   - **Adjust Minimum Gas Price**: Edit `~/.nolus/config/app.toml`:

     ```toml
     minimum-gas-prices = "0.0025unls"
     ```

   - **Configure Application State Pruning (Optional)**:

     Set `pruning = "everything"` or `pruning = "nothing"` in `~/.nolus/config/app.toml`.

## Set Up Cosmovisor (Optional)

1. **Install Cosmovisor**

   ```bash
   go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
   ```

2. **Add Environment Variables**

   Edit `~/.profile`:

   ```bash
   export DAEMON_NAME=nolusd
   export DAEMON_HOME=$HOME/.nolus
   export MONIKER_NAME="My First Nolus Node"
   ```

   Source the profile:

   ```bash
   source ~/.profile
   ```

3. **Adjust Folder Layout**

   Create the necessary directories:

   ```bash
   mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin
   mkdir -p $DAEMON_HOME/cosmovisor/upgrades
   ```

4. **Set Up Genesis Binary**

   Copy the Nolus node binary to the appropriate directory:

   ```bash
   cp /home/<your-user>/go/bin/nolusd $DAEMON_HOME/cosmovisor/genesis/bin
   ```

5. **Set Up a Service**

   Create a service file:

   ```bash
   sudo nano /etc/systemd/system/cosmovisor.service
   ```

   Add the following configuration:

   ```ini
   [Unit]
   Description=cosmovisor
   After=network-online.target

   [Service]
   User=<your-user>
   ExecStart=/home/<your-user>/go/bin/cosmovisor run start
   Restart=always
   RestartSec=3
   LimitNOFILE=4096
   Environment="DAEMON_NAME=nolusd"
   Environment="DAEMON_HOME=/home/<your-user>/.nolus"
   Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
   Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
   Environment="DAEMON_LOG_BUFFER_SIZE=512"

   [Install]
   WantedBy=multi-user.target
   ```

## Run Your Node

- **Without Cosmovisor**:

  ```bash
  nolusd start
  ```

- **With Cosmovisor**:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable cosmovisor
  sudo systemctl start cosmovisor
  ```

  Check the service status:

  ```bash
  sudo systemctl status cosmovisor
  ```

  View logs:

  ```bash
  journalctl -u cosmovisor -f
  ```

## Create a Validator

1. **Create a Local Key Pair**

   ```bash
   nolusd keys add <key-name>
   ```

   To restore an existing wallet:

   ```bash
   nolusd keys add <key-name> --recover
   ```

2. **Upgrade to a Validator**

   Ensure your node is fully synced before proceeding.

   ```bash
   nolusd tx staking create-validator \
   --amount 1000000unls \
   --commission-rate "0.05" \
   --commission-max-rate "0.10" \
   --commission-max-change-rate "0.01" \
   --min-self-delegation "1" \
   --pubkey=$(nolusd tendermint show-validator) \
   --moniker "<the-name-of-your-node>" \
   --chain-id "pirin-1" \
   --fees 500unls \
   --from <key-name>
   ```

3. **Confirm Your Validator Is Active**

   ```bash
   nolusd query tendermint-validator-set | grep "$(nolusd tendermint show-address)"
   ```

## Secure Your Keys

Make sure to secure your consensus key and application operator key.

## Restore a Validator

Replace the `~/.nolus/config/priv_validator.json` file and restart your node to restore your validator.

## Unjail a Validator

If jailed, submit the following command to unjail:

```bash
nolusd tx slashing unjail --from <yourKey> --chain-id pirin-1 --fees 500unls
```
