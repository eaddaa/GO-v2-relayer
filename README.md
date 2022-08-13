# GO-v2-relayer

# Setup GO Relayer v2 between Stride and GAIA
In current example we will learn how to set up GO Relayer v2 between two cosmos chains

## Preparation before you start
Before setting up relayer you need to make sure you already have:
1. Fully synchronized RPC nodes for each Cosmos project you want to connect
2. RPC enpoints should be exposed and available from relayer instance
#### RPC configuration is located in `config.toml` file
```
# STRIDE
laddr = "tcp://0.0.0.0:16657" in $HOME/.stride/config/config.toml
# GAIA
laddr = "tcp://0.0.0.0:23657" in $HOME/.gaia/config/config.toml  
```

3. Indexing is set to `kv` and is enabled on each node
```
# STRIDE
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.stride/config/config.toml
# GAIA
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.gaia/config/config.toml  
```

4. Dont forget to restart fullnode service after making changes
5. For each chain you will need to have wallets that are funded with tokens. This wallets will be used to do all relayer stuff

## Set up variables
All settings below are just example for IBC Relayer between `STRIDE-TESTNET-2` and `GAIA` testnets. Please fill with your own values.
```
RELAYER_ID='eaddaa#8291'            # use your Discord username here
STRIDE_RPC_ADDR='127.0.0.1:16657'    # use your own Stride RPC enpoint here
GAIA_RPC_ADDR='127.0.0.1:23657'      # use your own Gaia RPC enpoint here
```

## Update system
```
sudo apt update && sudo apt upgrade -y
```

## Install GO
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.3"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Install and initialize GO Relayer v2
```
git clone https://github.com/cosmos/relayer.git
cd relayer && git checkout v2.0.0-rc4
make install
rly config init --memo $RELAYER_ID
sudo mkdir $HOME/.relayer/chains
sudo mkdir $HOME/.relayer/paths
```

## Create relayer configuration files
### 1. Create stride chain json file
```
sudo tee $HOME/.relayer/chains/stride.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "STRIDE-TESTNET-2",
    "rpc-addr": "http://${STRIDE_RPC_ADDR}",
    "account-prefix": "stride",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.000ustrd",
    "gas": 200000,
    "timeout": "20s",
    "trusting-period": "8h",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

### 2. Create gaia chain json file
```
sudo tee $HOME/.relayer/chains/gaia.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "GAIA",
    "rpc-addr": "http://${GAIA_RPC_ADDR}",
    "account-prefix": "cosmos",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.000uatom",
    "gas": 200000,
    "timeout": "20s",
    "trusting-period": "8h",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

## Load chain configuration into the relayer
```
rly chains add --file=$HOME/.relayer/chains/stride.json stride
rly chains add --file=$HOME/.relayer/chains/gaia.json gaia
```

Check chains added to relayer
```
rly chains list
```

Successful output:
```
1: GAIA             -> type(cosmos) key(✘) bal(✘) path(✘)
2: STRIDE-TESTNET-2 -> type(cosmos) key(✘) bal(✘) path(✘)
```

## Load wallets into the relayer
```
rly keys restore stride wallet "your stride wallet mnemonic goes here"
rly keys restore gaia wallet "your gaia wallet mnemonic goes here"
```

Check wallet balance
```
rly q balance stride
rly q balance gaia
```

# Create stride-gaia paths json file
```
sudo tee $HOME/.relayer/paths/stride-gaia.json > /dev/null <<EOF
{
  "src": {
    "chain-id": "STRIDE-TESTNET-2",
    "client-id": "07-tendermint-0",
    "connection-id": "connection-0"
  },
  "dst": {
    "chain-id": "GAIA",
    "client-id": "07-tendermint-0",
    "connection-id": "connection-0"
  },
  "src-channel-filter": {
    "rule": "allowlist",
    "channel-list": ["channel-0", "channel-1", "channel-2", "channel-3", "channel-4"]
  }
}
EOF
```

# add paths
```
rly paths add STRIDE-TESTNET-2 GAIA stride-gaia --file $HOME/.relayer/paths/stride-gaia.json
```

Check path is correct
```
rly paths list
```

Successful output:
```
0: stride-gaia -> chns(✔) clnts(✔) conn(✔) (STRIDE-TESTNET-2<>GAIA)
```
If everything is correct, we can proceed with relayer service creation



## Create service
```
sudo tee /etc/systemd/system/relayerd.service > /dev/null <<EOF
[Unit]
Description=GO Relayer v2 Service
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which rly) start stride-gaia -p events
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## If you run hermesd as your main relayer, please stop it before running GO v2 Relayer on the next step.
```
sudo systemctl stop hermesd
sudo systemctl disable hermesd
```

## Start the service
```
sudo systemctl daemon-reload
sudo systemctl enable relayerd
sudo systemctl start relayerd
```

## Check logs
```
journalctl -u relayerd -f -o cat
```

# Submitting the task

## 1. Get relayer transaction hash
Navigate [chain explorer](https://poolparty.stride.zone/STRIDE/) and check your wallet transaction history to find the `UpdateClient` and `RecvPacket` messages. You will need those to submit the form



2.2. Create new files in your forked repository
```
relayer/configs/stride/chains/gaia.json
relayer/configs/stride/chains/stride.json
relayer/configs/stride/paths/stride-gaia.json
```

2.3. Fill up the [submission form](https://forms.gle/urhJDEkqfMM9h1367) with links to your relayer transactions and link to your forked repository

# Usefull commands

## Completely remove relayer
> NOTE: This will delete your relayer from the machine!
```
sudo systemctl stop relayerd
sudo systemctl disable relayerd
sudo rm /etc/systemd/system/relayerd* -rf
sudo rm $(which rly) -rf
sudo rm -rf $HOME/.relayer
sudo rm -rf $HOME/relayer



```
