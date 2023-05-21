<h1 align="center"> Andromeda </h1>

> Neden Andromeda'ya katıldım? Boşta sunucum vardı, ödül `potansiyelli` projeleri kuruyorum.

> Andromeda ekibi, ilk 2 testnetine incentivize mi sorulduğunda hayır cevabını veriyordu, fakat `galileo-3` için asla cevap vermiyorlar. Discord kanallarına geçen yıl Eylül ayında katılmıştım, ticket'tan konuştuğumda yine bu şekilde cevaplar alıyordum.

> Andromeda'ya katılmamın bir diğer nedeni sevdiğim proje ve yayınlarımda Luna'dan Cosmos'a geçmesini beklediğim bir proje var dediğim proje Andromeda idi. `Boş sunucuma kurdum takılsın orda.`

<h1 align="center"> Donanım </h1>

> Hetzner'den 3 dolarlık sunucuya kurdum. Hetzner üye olduğunuzda 20 dolarlık sunucu veriyor: [Link.](https://hetzner.cloud/?ref=gIFAhUnYYjD3) Ödülü net olmayan projeler için kullanabilirsiniz, ücretsiz zaten üye olmanız yetiyor.

```sh
# Benim kurduğum (şimdilik yetiyor):
2 CPU 2 RAM

# Kesin yetecek donanım:
3 CPU 4 RAM
```

<h1 align="center"> Sunucu güncellemesi </h1>

```sh
# Gerekli portlar
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656
sudo ufw allow 26657

# Sunucu update 
sudo apt update && sudo apt upgrade -y
sudo apt install git
sudo apt install -y curl git jq lz4 build-essential unzip

# Go'yu yüklüyoruz
if [ "$(go version)" != "go version go1.20.2 linux/amd64" ]; then \
ver="1.20.2" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile ; \
fi

go version

# go version sonrası çıktı: go version go1.20.2 linux/amd64
```
<h1 align="center"> Andromeda'yı yüklüyoruz </h1>

```sh
NODE_MONIKER="Validatör İsmin"

cd || return
rm -rf andromedad
git clone https://github.com/andromedaprotocol/andromedad.git
cd andromedad || return
git checkout galileo-3-v1.1.0-beta1

make install

andromedad version 
# version çıktı: galileo-3-v1.1.0-beta1

andromedad config keyring-backend test
andromedad config chain-id galileo-3
andromedad init "$NODE_MONIKER" --chain-id galileo-3
```
<h1 align="center"> Addrbook, Seeds ve Peers ayarlıyoruz </h1>

```sh
curl -s https://raw.githubusercontent.com/andromedaprotocol/testnets/galileo-3/genesis.json > $HOME/.andromedad/config/genesis.json
curl -s https://snapshots2-testnet.nodejumper.io/andromeda-testnet/addrbook.json > $HOME/.andromedad/config/addrbook.json

SEEDS=""
PEERS=""
sed -i 's|^seeds *=.*|seeds = "'$SEEDS'"|; s|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.andromedad/config/config.toml

sed -i 's|^pruning *=.*|pruning = "custom"|g' $HOME/.andromedad/config/app.toml
sed -i 's|^pruning-keep-recent  *=.*|pruning-keep-recent = "100"|g' $HOME/.andromedad/config/app.toml
sed -i 's|^pruning-interval *=.*|pruning-interval = "10"|g' $HOME/.andromedad/config/app.toml
sed -i 's|^snapshot-interval *=.*|snapshot-interval = 0|g' $HOME/.andromedad/config/app.toml

sed -i 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.0001uandr"|g' $HOME/.andromedad/config/app.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.andromedad/config/config.toml
```

<h1 align="center"> Servis dosyası </h1>

```sh
# Komple kopyalayarak girebilirsiniz

sudo tee /etc/systemd/system/andromedad.service > /dev/null << EOF
[Unit]
Description=Andromeda testnet Node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which andromedad) start
Restart=on-failure
RestartSec=10
LimitNOFILE=10000
[Install]
WantedBy=multi-user.target
EOF
```
<h1 align="center"> Snapshot ve Hızlı Sync Olmak </h1>

```sh
andromedad tendermint unsafe-reset-all --home $HOME/.andromedad --keep-addr-book

SNAP_NAME=$(curl -s https://snapshots2-testnet.nodejumper.io/andromeda-testnet/info.json | jq -r .fileName)
curl "https://snapshots2-testnet.nodejumper.io/andromeda-testnet/${SNAP_NAME}" | lz4 -dc - | tar -xf - -C "$HOME/.andromedad"

sudo systemctl daemon-reload
sudo systemctl enable andromedad
sudo systemctl start andromedad

sudo journalctl -u andromedad -f --no-hostname -o cat
# Akan ekranda Height mevcut bloğunuzdur, explorer'dan latest block'a ulaşabilirsiniz.
```

<h1 align="center"> Sync olurken cüzdan ve token oluşturalım </h1>

```sh
# Cüzdan oluşturma
andromedad keys add rues
# rues kısmına wallet isminizi girin ve kaydedin bilgilerinizi.

# Recover yapmak isterseniz
andromedad keys add rues --recover
# Cüzdan bilgileriniz ile faucetten token alın, discorda mevcut.

# Token geldiğini kontrol edelim
andromedad query bank balances cüzdan_adresiniz
# Tokenleri hangi blokta talep ederseniz, node'unuz o bloğa gelene kadar tokenlerinizi göstermez
# Güncel blokta değilseniz explorer'dan bakınız.

# Bu komuttan sonra false çıktısı alırsanız validatör oluşturma kısmına geçebilirsiniz.
andromedad status 2>&1 | jq .SyncInfo.catching_up
```
## Validator oluşturma
```sh
andromedad tx staking create-validator \
--amount=1000000uandr \
--pubkey=$(andromedad tendermint show-validator) \
--moniker="$NODE_MONIKER" \
--chain-id=galileo-3 \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=10000uandr \
--from=rues \
-y
```











































