Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 4GB RAM, 100GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.19.3"
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
echo "export UPTICK_CHAIN_ID="origin_1170-3"" >> $HOME/.bash_profile
echo "export UPTICK_PORT="10"" >> $HOME/.bash_profile
source $HOME/.bash_profile
``` 
**download binary**
```
cd $HOME
rm -rf uptick
git clone https://github.com/UptickNetwork/uptick.git
cd uptick
git checkout v0.2.19
make install
```

**config and init app**
```
uptickd config node tcp://localhost:${UPTICK_PORT}657
uptickd config keyring-backend os
uptickd config chain-id origin_1170-3
uptickd init "test" --chain-id origin_1170-3
```

**download genesis and addrbook**
```
wget -O $HOME/.uptickd/config/genesis.json https://server-4.itrocket.net/testnet/uptick/genesis.json
wget -O $HOME/.uptickd/config/addrbook.json  https://server-4.itrocket.net/testnet/uptick/addrbook.json
```

# set seeds and peers
SEEDS="cc4d304a2d2062dfe9f77a7ee0b9aa3aa1ba7869@uptick-testnet-seed.itrocket.net:10656"
PEERS="b232825b5b50857c13d2e89a2988eee462ed5462@uptick-testnet-peer.itrocket.net:10656,653994d33ea77a6e3dcadbe2fe8b9a58177e3573@57.128.22.78:21456,0b442a23cc0e5b27877e1e4fd040c3155866ab12@207.180.236.118:26656,8a64818c94e5b1aca7974a658c58ca9248c8b8fe@207.180.231.123:26656,cda6bd82e62e8c91b54498d7fbd930b962f1125b@47.128.211.171:26656,ac6ae007ce6ad5cd5df15ac74edafcf3a9f6f7c9@190.2.146.152:29656,6dec50675cfd4b625694483259476cb41a9a3956@157.90.33.62:22656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.uptickd/config/config.toml


# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${UPTICK_PORT}317%g;
s%:8080%:${UPTICK_PORT}080%g;
s%:9090%:${UPTICK_PORT}090%g;
s%:9091%:${UPTICK_PORT}091%g;
s%:8545%:${UPTICK_PORT}545%g;
s%:8546%:${UPTICK_PORT}546%g;
s%:6065%:${UPTICK_PORT}065%g" $HOME/.uptickd/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${UPTICK_PORT}658%g;
s%:26657%:${UPTICK_PORT}657%g;
s%:6060%:${UPTICK_PORT}060%g;
s%:26656%:${UPTICK_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${UPTICK_PORT}656\"%;
s%:26660%:${UPTICK_PORT}660%g" $HOME/.uptickd/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.uptickd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.uptickd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.uptickd/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "10000000000auoc"|g' $HOME/.uptickd/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.uptickd/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.uptickd/config/config.toml

# create service file
sudo tee /etc/systemd/system/uptickd.service > /dev/null <<EOF
[Unit]
Description=Uptick node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.uptickd
ExecStart=$(which uptickd) start --home $HOME/.uptickd --chain-id origin_1170-3
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
uptickd tendermint unsafe-reset-all --home $HOME/.uptickd
if curl -s --head curl https://server-4.itrocket.net/testnet/uptick/uptick_2024-08-25_3302817_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/uptick/uptick_2024-08-25_3302817_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.uptickd
    else
  echo "no snapshot founded"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable uptickd
sudo systemctl restart uptickd && sudo journalctl -u uptickd -f
Automatic Installation
pruning: custom: 100/0/10 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/uptick/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
uptickd keys add $WALLET

# to restore exexuting wallet, use the following command
uptickd keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(uptickd keys show $WALLET -a)
VALOPER_ADDRESS=$(uptickd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
uptickd status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
uptickd query bank balances $WALLET_ADDRESS 
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, auoc
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
uptickd tx staking create-validator \
--amount 1000000auoc \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(uptickd tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id origin_1170-3 \
--fees 3000000000000000auoc \
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
sudo ufw allow ${UPTICK_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop uptickd
sudo systemctl disable uptickd
sudo rm -rf /etc/systemd/system/uptickd.service
sudo rm $(which uptickd)
sudo rm -rf $HOME/.uptickd
sed -i "/UPTICK_/d" $HOME/.bash_profile
