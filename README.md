Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export TELLOR_CHAIN_ID="layertest-2"" >> $HOME/.bash_profile
echo "export TELLOR_PORT="54"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf layer
git clone https://github.com/tellor-io/layer
cd layer
git checkout v0.7.1-fix
go build ./cmd/layerd
mv layerd $HOME/go/bin
```

**config and init app**
```
layerd init "itrocket_rpc" --chain-id layertest-2
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${TELLOR_PORT}657\"|" $HOME/.layer/config/client.toml
sed -i -e "s|^keyring-backend *=.*|keyring-backend = \"test\"|" $HOME/.layer/config/client.toml
sed -i -e "s|^chain-id *=.*|chain-id = \"layertest-1\"|" $HOME/.layer/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.layer/config/genesis.json https://server-5.itrocket.net/testnet/tellor/genesis.json
wget -O $HOME/.layer/config/addrbook.json  https://server-5.itrocket.net/testnet/tellor/addrbook.json
```

**set seeds and peers**
```
SEEDS="f9e5273c92ee8635c81100f00c49d11068fef853@tellor-testnet-seed.itrocket.net:19656"
PEERS="0e9c659ff5ec9d226b6952d5b42e36daf1b13485@tellor-testnet-peer.itrocket.net:54656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.layer/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${TELLOR_PORT}317%g;
s%:8080%:${TELLOR_PORT}080%g;
s%:9090%:${TELLOR_PORT}090%g;
s%:9091%:${TELLOR_PORT}091%g;
s%:8545%:${TELLOR_PORT}545%g;
s%:8546%:${TELLOR_PORT}546%g;
s%:6065%:${TELLOR_PORT}065%g" $HOME/.layer/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${TELLOR_PORT}658%g;
s%:26657%:${TELLOR_PORT}657%g;
s%:6060%:${TELLOR_PORT}060%g;
s%:26656%:${TELLOR_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${TELLOR_PORT}656\"%;
s%:26660%:${TELLOR_PORT}660%g" $HOME/.layer/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.layer/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.layer/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.layer/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "1loya"|g' $HOME/.layer/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.layer/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.layer/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/layerd.service > /dev/null <<EOF
[Unit]
Description=Tellor node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.layer
ExecStart=$(which layerd) start --home $HOME/.layer --api.enable --api.swagger --key-name test --keyring-backend test
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
layerd tendermint unsafe-reset-all --home $HOME/.layer
if curl -s --head curl https://server-5.itrocket.net/testnet/tellor/tellor_2024-10-04_1884527_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/tellor/tellor_2024-10-04_1884527_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.layer
    else
  echo "no snapshot founded"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable layerd
sudo systemctl restart layerd && sudo journalctl -u layerd -f
Automatic Installation
pruning: custom: 100/0/50 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/tellor/autoinstall/)
Create wallet
```

**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
layerd keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
layerd keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(layerd keys show $WALLET -a)
VALOPER_ADDRESS=$(layerd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
layerd status 2>&1 | jq 
```

**before creating a validator, you need to fund your wallet and check balance**
```
layerd query bank balances $WALLET_ADDRESS
```

Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.layer/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://tellor-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, arkh
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(layerd comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000arkh\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
layerd tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id layertest-2 \
	--gas auto --gas-adjustment 1.5 \
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${TELLOR_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop layerd
sudo systemctl disable layerd
sudo rm -rf /etc/systemd/system/layerd.service
sudo rm $(which layerd)
sudo rm -rf $HOME/.layer
sed -i "/TELLOR_/d" $HOME/.bash_profile
