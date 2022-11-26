# Tgrade: elizabeth-1 Long Term Testnet

Elizabeth-1 is our long term testnet.  It is an enviroment for the community to build and test their Trusted Circle and applications on.

## Hardware Requirements
For running a tgrade validator. We tested successfully with the following Architecture:

- Ubuntu 20.04 LTS
- go version 1.18.1 [1] or newer
- Installed packages make and build-essential [2] [3]
- 2 or more CPU cores Intel or AMD chipset
- At least 40GB of disk storage
- At least 4GB of memory (RAM)

## Prerequists

Update Go to a recent version 

```
sudo rm -rf /usr/local/go && cd ~/ && \
sudo wget https://go.dev/dl/go1.19.1.linux-amd64.tar.gz && \
sudo tar -C /usr/local -xvf go1.19.1.linux-amd64.tar.gz && \
sudo chown -R $USER:$USER /usr/local/go && rm -f go1.19.1.linux-amd64.tar.gz
```
Add the following to the end of your `.bashrc` and run

```
nano ~/.bashrc
```
```
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
```
```
source .bashrc
```

## Build the tgrade binary
The tgrade binary is the backbone of the platform. 
```
cd ~
git clone https://github.com/confio/tgrade
cd tgrade
git checkout v2.0.2
make build
```

Move the binary to an executable path
```
sudo mv build/tgrade /usr/bin
```
Check that you have the right `tgrade` version installed
```
tgrade version --long
```
This should return:
```
name: tgrade
server_name: tgrade
version: 2.0.2
commit: a990225d4b92f09d7b6fad3d6147d71c3711ece4
build_tags: netgo,ledger
go: go version go1.19.1 linux/amd64
```

## Setting up a Testnet Genesis Tgrade Validator - PHASE 1

### Initialize your genesis and configuration files
Initialize your genesis and configuration files for your validator node

Usage:
```
tgrade init [moniker] [flags]
```
Download the pre-genesis genesis.json:

```
cd $HOME/.tgrade/config
rm genesis.json
wget https://github.com/lightiv/tgrade/raw/main/Elizabeth-1/gentx_genesis.json
mv gentx_genesis.json genesis.json
```
```
tgrade tendermint unsafe-reset-all --home $HOME/.tgrade
tgrade init "<NAME_OF_MY_VALIDATOR>" --chain-id elizabeth-1
```

Create an Address
```
tgrade keys add <KEY_NAME> 
```

If using a Ledger Nano (Very much optional for testnet)
```
tgrade keys add <KEY_NAME> --ledger
```
## ***It is critical that you stop now and backup the mnemonic just created.*** ##  
  
This is the only way to recover your validator key/address.  Additionally you need to back up the following files now.  This is your validator's identity.
```
$HOME/.tgrade/config/node_key.json (Not mandatory.)
$HOME/.tgrade/config/priv_validator_key.json **(Critical!!!)**
```

## Create the genesis transaction (gentx) - PHASE 2

Add your account to your local genesis file with a given amount and the key you just created. Use only 1000000000utgd, other amounts will be ignored.
```
tgrade add-genesis-account $(tgrade keys show <KEY_NAME> -a) 1000000000utgd
```
Create the gentx, use only `90000000000utgd`:
```
tgrade gentx <KEY_NAME> 90000000000utgd --chain-id=elizabeth-1
```
If all goes well, you will see a message similar to the following:
```
Genesis transaction written to "${HOME}/.tgrade/config/gentx/gentx-******.json"
```
Change minimum gas prices in `${HOME}/.tgrade/config/app.toml` to `0.0025utgd`.

### Submit the Genesis transaction

- Fork the testnets repo into your Github account

- Clone your repo using
  ```
  git clone https://github.com/<your-github-username>/testnets
  ```
- Fork `the testnets repo`
- Clone your repor using
  ```
  git clone https://github.com/<your-github-username>/testnets
  ```
- Copy the generated gentx json file to <repo_path>/elizabeth-1/gentx/. If you want, rename it to <NAME_OF_MY_VALIDATOR>.json so it's easier to identify.
  ```
  cd testnets
  cp "${HOME}/.tgrade/config/gentx/gentx*.json" ./elizabeth-1/gentx/
  ```
- Commit and Push to your forked repo
- Create a PR from your online repo

**Gentxs will be accepted until <DATE_TIME_UTC>**

## Start You Validator - PHASE 3

### Get the final genesis file
```
wget https://raw.githubusercontent.com/confio/tgrade-networks/main/elizabeth-1/config/genesis.json -O ~/.tgrade/config/genesis.json
```
Verify Genesis sha256 hash
```
sha256sum "${HOME}/.tgrade/config/genesis.json"
# f14bf43d43e69d470859d2ba1e1eee6d576229aeb1e7c26cc98d29254886820a
```

### Start syncing

Add peers to the `persistent_peers =` in .tgrade/config/config.toml
```
nano .tgrade/config/config.toml
```
**Open the p2p firewall port for optimal network performance**
```
sudo ufw allow from any to any port 26656 [<- or custom port] proto tcp
```

Create a service to run tgrade
```
sudo nano /lib/systemd/system/tgrade.service
```
Add the following
```
Description=Tgrade daemon
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/bin/tgrade start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```
Enable, start, and monitor the service
```
sudo systemctl enable tgrade
sudo systemctl restart tgrade; sudo journalctl --no-hostname -fu tgrade -o cat
```
