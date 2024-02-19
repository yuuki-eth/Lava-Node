# Lava-Node

VPS(Virtual Private Server)
U can buy it from
🔶Contabo(If u have Credit-Card or Debit-Card then use the below👇)
🔷Open link: https://contabo.com/en/


 Buy the Suitable VPS and then open ur VPS in MOBILE SSH App(if u use Iphone/Android), use Putty(PC)

🌟Run a full node🌟

Install Dependencies
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential fail2ban ufw
sudo apt -qy upgrade
Configure Moniker
MONIKER="<your-moniker-name>"
Install Go
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
📌Before proceeding, ensure u visit the Lava GitHub repository to verify the latest version. Look on the right-hand side of the page to determine the most recent release. Once u’ve identified the latest version, update the command “git checkout” accordingly to reflect this newest version.

Build Binaries
cd $HOME
rm -rf lava
git clone https://github.com/lavanet/lava.git
cd lava
git checkout v0.35.1

# Build binaries
export LAVA_BINARY=lavad
make build

# Prepare binaries for Cosmovisor
mkdir -p $HOME/.lava/cosmovisor/genesis/bin
mv build/lavad $HOME/.lava/cosmovisor/genesis/bin/
rm -rf build

# Create application symlinks
sudo ln -s $HOME/.lava/cosmovisor/genesis $HOME/.lava/cosmovisor/current -f
sudo ln -s $HOME/.lava/cosmovisor/current/bin/lavad /usr/local/bin/lavad -f
Cosmovisor Setup & Create Service
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

# Create service
sudo tee /etc/systemd/system/lava.service > /dev/null << EOF
[Unit]
Description=lava node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.lava"
Environment="DAEMON_NAME=lavad"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable lava.service
Initialize the node
# Set node configuration
lavad config chain-id lava-testnet-2
lavad config keyring-backend test
lavad config node tcp://localhost:14457

# Initialize the node
lavad init $MONIKER --chain-id lava-testnet-2

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/lava-testnet/genesis.json > $HOME/.lava/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/lava-testnet/addrbook.json > $HOME/.lava/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@lava-testnet.rpc.kjnodes.com:14459\"|" $HOME/.lava/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0ulava\"|" $HOME/.lava/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.lava/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:14458\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:14457\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:14460\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:14456\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":14466\"%" $HOME/.lava/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:14417\"%; s%^address = \":8080\"%address = \":14480\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:14490\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:14491\"%; s%:8545%:14445%; s%:8546%:14446%; s%:6065%:14465%" $HOME/.lava/config/app.toml
Download latest chain snapshot
curl -L https://snapshots.kjnodes.com/lava-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.lava
[[ -f $HOME/.lava/data/upgrade-info.json ]] && cp $HOME/.lava/data/upgrade-info.json $HOME/.lava/cosmovisor/genesis/upgrade-info.json
Start service and check the logs
sudo systemctl start lava.service && sudo journalctl -u lava.service -f --no-hostname -o cat
📌After getting synced then u can proceed to next step

Become a Validator
To initiate the journey of becoming a validator, you’ll need to follow these steps.

Create a new wallet
lavad keys add wallet
📌Ensure to save the Address, private key or phrases

Get Faucets from the Lava_Network DC
To receive funds, visit the #faucet channel on the DC. Here, u can request faucet by sharing the address u previously created. Once in the channel, submit ur request by typing $request followed by ur address. For instance, u would type this to request funds for that specific address:
$request {Address}

u can check ur wallet balance using this command:

lavad q bank balances $(lavad keys show wallet -a)
Save the private validator key
Now save the private validator key, since it’s crucial to secure this file adequately to maintain the integrity and security of ur validator operations.

cat $HOME/.lava/config/priv_validator_key.json
Create the Validator
Check the network synchronization status

lavad status 2>&1 | jq .SyncInfo.catching_up
📌As soon as the value of catching_up becomes “false”, u can proceed to the final step and create ur validator

# Make sure you have adjusted **moniker**, **details** and **website** to match your values.

lavad tx staking create-validator \
--amount=150000ulava \
--pubkey=$(lavad tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id=lava-testnet-2 \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=10000ulava \
--from=wallet \
-y
Now make sure you see the validator details

lavad q staking validator $(lavad keys show wallet --bech val -a)
Congratulations👏! u’ve successfully become a validator on the Lava Network. the validator should appear in the list at https://lava.explorers.guru/validators (u can find it by searching for the wallet address or ur Moniker, and then see it in the delegations to ur validator on this wallet)

delegate these tokens to ur node by using the following command

lavad tx staking delegate YOUR_NODE_OPERATOR_ADRESS_HERE 1000000ulava --from wallet --chain-id lava-testnet-2
THAT’S IT FOR NOW!!!!😒😒
I hope you have found this thread 🧵 helpful.
Thnx for reading my thread.
Stay tuned for more HeHe boy😋😂😁
Follow me & join our channel
@cipher_airdrop
for more awesome strategies.
CLAP🌟

Disclaimer: Use of Lava Node Setup Guide
The information provided in this guide is intended for educational and informational purposes only. The setup procedures outlined here are based on the author’s understanding and experience, and they may not represent the only or the best methods for configuring a Lava node.

Before implementing any changes to your Lava node configuration, it is strongly recommended to review and understand the potential implications of each step. The user is solely responsible for the consequences of any actions taken based on the information provided in this guide.

The cryptocurrency and blockchain landscape is dynamic, and updates to software and protocols may occur. Users are encouraged to consult official documentation, forums, or community channels for the latest and most accurate information regarding Lava node setup and configuration.

Additionally, users should exercise caution and adhere to best security practices when dealing with cryptocurrency nodes. This includes implementing secure access controls, regularly updating software, and keeping private keys secure.

The guide author and associated entities are not liable for any damages, losses, or other consequences that may arise from following the procedures outlined in this guide. Users are advised to proceed with caution and seek professional advice if needed.

By using this guide, you acknowledge and agree to the terms of this disclaimer.
