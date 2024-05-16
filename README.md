# Entrypoint

### Entrypoint node Installation Instructions.

[Official documentation](https://docs.entrypoint.zone)

System requirements:</br>
CPU: Quad Core or larger AMD or Intel (amd64) CPU
RAM:32GB RAM
SSD:1TB NVMe Storage
100MBps bidirectional internet connection
OS: Ubuntu 20.04 or 22.04</br>

You can take a weaker server

### Network configurations: </br>
Outbound - allow all traffic </br>
Inbound - open the following ports :</br>
1317 - REST </br>
26657 - TENDERMINT_RP </br>
26656 - Cosmos </br>
Running as a Provider? Add these specific ports: </br>
22221 - provider port </br>
22231 - provider port </br>
22241 - provider port </br>

### Installing the Babylon Node

1. Preparing the server/Required packages installation</br>
```
sudo apt update
sudo apt upgrade -y
sudo apt-get install libclang-dev
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```
### Go installation.
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile
go version
```

### Download and build binaries
```
cd $HOME
curl -s https://github.com/entrypoint-zone/testnets/releases/download/v1.3.0/entrypointd-1.3.0-linux-amd64 > entrypointd
chmod +x entrypointd
mkdir -p $HOME/go/bin/
mv entrypointd $HOME/go/bin/
```

# Config and init app
```
entrypointd config chain-id entrypoint-pubtest-2
entrypointd config keyring-backend test
entrypointd config node tcp://localhost:26657
entrypointd init "your moniker" --chain-id entrypoint-pubtest-2
```

# Download genesis and addrbook
```
curl -L https://snapshots-testnet.nodejumper.io/entrypoint-testnet/genesis.json > $HOME/.entrypoint/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/entrypoint-testnet/addrbook.json > $HOME/.entrypoint/config/addrbook.json
```

# Set seeds and peers
```
sed -i -e 's|^seeds *=.*|seeds = "e1b2eddac829b1006eb6e2ddbfc9199f212e505f@entrypoint-testnet-seed.itrocket.net:34656,7048ee28300ffa81103cd24b2af3d1af0c378def@entrypoint-testnet-peer.itrocket.net:34656,05419a6f8cc137c4bb2d717ed6c33590aaae022d@213.133.100.172:26878,f7af71e7f32516f005192b21f1a83ca3f4fef4da@142.132.202.92:32256"|' $HOME/.entrypoint/config/config.toml
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.01ibc/8A138BC76D0FB2665F8937EC2BF01B9F6A714F6127221A0E155106A45E09BCC5"|' $HOME/.entrypoint/config/app.toml
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.entrypoint/config/app.toml
```

# Create service file
```
sudo tee /etc/systemd/system/entrypointd.service > /dev/null << EOF
[Unit]
Description=EntryPoint node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which entrypointd) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable entrypointd.service
```

# Reset and download snapshot
```
curl "https://snapshots-testnet.nodejumper.io/entrypoint-testnet/entrypoint-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.entrypoint"
```

# enable and start service
```
sudo systemctl start entrypointd.service
sudo journalctl -u entrypointd.service -f --no-hostname -o cat
```

### Becoming a Validator

# Create wallet key new
```
entrypointd keys add wallet
```

(OPTIONAL) RECOVER EXISTING KEY
```
entrypointd keys add wallet --recover
```

# check sync status, once your node is fully synced, the output from above will print "false"
```
entrypointd status 2>&1 | jq -r '.SyncInfo.catching_up // .sync_info.catching_up'
```

### We receive tokens from the tap in the [discord](https://discord.gg/ukdTEH4S8F)
```
#uentry-token-faucet
```

# before creating a validator, you need to fund your wallet and check balance
```
entrypointd q bank balances $(entrypointd keys show wallet -a) 
```
# Create validator
```
entrypointd tx staking create-validator \
--amount=1000000uentry \
--pubkey=$(entrypointd tendermint show-validator) \
--moniker="Moniker" \
--identity=FFB0AA51A2DF5955 \
--details="I love YTWO❤️" \
--chain-id=entrypoint-pubtest-2 \
--commission-rate=0.10 \
--commission-max-rate=0.20 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=wallet \
--gas-prices=0.01ibc/8A138BC76D0FB2665F8937EC2BF01B9F6A714F6127221A0E155106A45E09BCC5 \
--gas-adjustment=1.5 \
--gas=auto \
-y 
```

### Update
```
No update

Current network:entrypoint-pubtest-2
Current version:v1.3.0
```

### Useful commands

Check balance
```
entrypointd q bank balances $(entrypointd keys show wallet -a) 
```

CHECK SERVICE LOGS
```
sudo journalctl -u entrypointd -f --no-hostname -o cat
```

RESTART SERVICE
```
sudo systemctl restart entrypointd
```

GET VALIDATOR INFO
```
entrypointd status 2>&1 | jq -r '.ValidatorInfo // .validator_info'
```

DELEGATE TOKENS TO YOURSELF
```
entrypointd tx staking delegate $(entrypointd keys show wallet --bech val -a) 1000000uentry --from wallet --chain-id entrypoint-pubtest-2 --gas-prices 0.01ibc/8A138BC76D0FB2665F8937EC2BF01B9F6A714F6127221A0E155106A45E09BCC5 --gas-adjustment 1.5 --gas auto -y 
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
sudo systemctl stop entrypointd && sudo systemctl disable entrypointd && sudo rm /etc/systemd/system/entrypointd.service && sudo systemctl daemon-reload && rm -rf $HOME/.entrypoint && rm -rf testnets && sudo rm -rf $(which entrypointd) 
```
