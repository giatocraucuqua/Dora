Manual Installation
Official Documentation
Recommended Hardware: 8 Cores, 32GB RAM, 500GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.20.3"
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
echo "export DORA_CHAIN_ID="vota-testnet"" >> $HOME/.bash_profile
echo "export DORA_PORT="43"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf doravota
git clone https://github.com/DoraFactory/doravota.git
cd doravota
git checkout 0.4.1
make install
```

**config and init app**
```
dorad config node tcp://localhost:${DORA_PORT}657
dorad config keyring-backend os
dorad config chain-id vota-testnet
dorad init "test" --chain-id vota-testnet
```

**download genesis and addrbook**
```
wget -O $HOME/.dora/config/genesis.json https://server-4.itrocket.net/testnet/dora/genesis.json
wget -O $HOME/.dora/config/addrbook.json  https://server-4.itrocket.net/testnet/dora/addrbook.json
```

**set seeds and peers**
```
SEEDS="dcf2d8132162293de52ca474eee308a03689ef75@dora-testnet-seed.itrocket.net:39656"
PEERS="2ae884cc092d593207cccfb16e81313ef0b478f4@dora-testnet-peer.itrocket.net:43656,ef37c0e26ed44bd6f628638be7009ceb229801e2@65.108.206.118:26656,1c070c5b38c716370af9b0d344209821cceeceb3@18.142.245.25:26656,4f9205cb2c34541f7daebe2a43f303c0d758b383@5.75.133.134:43656,1a162fdf46cabfc9b63b1525c327e101f9418329@65.109.30.13:26757,6bedeb292ec261dc9664233c1e9f43d88657ce2c@167.235.178.134:25356,3c3aa7ede95c8e1391331f3970b62b4ca4003a12@15.235.212.150:56096,635bc8a8ff9d6bc15cea091a27bcd01578ddffa2@65.109.62.39:43656,e486efcbdddf9b496bcf76ed8bc216977929ae6a@188.40.66.173:25356,bfde287f9659ac66d04b5aad68c37658b821e2e5@34.48.66.38:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.dora/config/config.toml
```
**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${DORA_PORT}317%g;
s%:8080%:${DORA_PORT}080%g;
s%:9090%:${DORA_PORT}090%g;
s%:9091%:${DORA_PORT}091%g;
s%:8545%:${DORA_PORT}545%g;
s%:8546%:${DORA_PORT}546%g;
s%:6065%:${DORA_PORT}065%g" $HOME/.dora/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${DORA_PORT}658%g;
s%:26657%:${DORA_PORT}657%g;
s%:6060%:${DORA_PORT}060%g;
s%:26656%:${DORA_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${DORA_PORT}656\"%;
s%:26660%:${DORA_PORT}660%g" $HOME/.dora/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.dora/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.dora/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.dora/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "100000000000peaka"|g' $HOME/.dora/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.dora/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.dora/config/config.toml
```
**create service file**
```
sudo tee /etc/systemd/system/dorad.service > /dev/null <<EOF
[Unit]
Description=Dora node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.dora
ExecStart=$(which dorad) start --home $HOME/.dora
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
dorad tendermint unsafe-reset-all --home $HOME/.dora
if curl -s --head curl https://server-4.itrocket.net/testnet/dora/dora_2025-02-19_5991366_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/dora/dora_2025-02-19_5991366_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.dora
    else
  echo "no snapshot found"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable dorad
sudo systemctl restart dorad && sudo journalctl -u dorad -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/dora/autoinstall/)
```

Create wallet
**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
dorad keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
dorad keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(dorad keys show $WALLET -a)
VALOPER_ADDRESS=$(dorad keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
dorad status 2>&1 | jq 
```

**before creating a validator, you need to fund your wallet and check balance**
```
dorad query bank balances $WALLET_ADDRESS
```

**Node Sync Status Checker**
```
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.dora/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://dora-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

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
```

Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, peaka
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
dorad tx staking create-validator \
--amount 1000000peaka \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(dorad tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id vota-testnet \
--fees 20000000000000000peaka \
-y
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
sudo ufw allow ${DORA_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop dorad
sudo systemctl disable dorad
sudo rm -rf /etc/systemd/system/dorad.service
sudo rm $(which dorad)
sudo rm -rf $HOME/.dora
sed -i "/DORA_/d" $HOME/.bash_profile
