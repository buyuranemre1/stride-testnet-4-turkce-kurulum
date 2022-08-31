# STRIDE-TESTNET-4 Türkçe Kurulum Rehberi
Description of how to install stride testnet on your server (in Turkish)
# Stride node setup - STRIDE-TESTNET-4
Resmi Dökümantasyon:

> Validator kurulum talimatları:

https://github.com/Stride-Labs/testnet

> Explorer (Tarayıcı):

https://poolparty.stride.zone/

> Network Chain ID

STRIDE-TESTNET-4

> Denom: 

ustrd

## Donanım Gereksinimleri

Cosmos ekosistemindeki diğer ağlar gibi donanım gereksinimleri oldukça kolay.

`Tavsiye Edilen Donanım Gereksinimleri`

- 4x CPUs; Olabildiği kadar hızlı CPU avantajdır
- 8GB RAM
- 200GB of storage (SSD veya NVME)
- Stabil bir interet bağlantısı (Mainnet süresince trafik oldukça minimal olacaktır; 10Mbps yeterlidir - Launch'tan sonra en az 100Mbps beklenir.)

## Sunucuyu hazırlıyoruz

Repoları güncelleyelim
```
sudo apt update && sudo apt upgrade -y
```
Gerekli kütüphaneleri yükleyelim
```
sudo apt install curl build-essential git wget jq make gcc tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y
```
Test edelim
```
bash <(wget -qO- -o /dev/null yabs.sh)
```
Sadece diskleri test edelim
```
bash <(wget -qO- -o /dev/null yabs.sh) -ig
```
Sadece internet hızını kontrol edelim
```
bash <(wget -qO- -o /dev/null yabs.sh) -dg
```
Sadece sistem performansını test edelim
```
bash <(wget -qO- -o /dev/null yabs.sh) -di
```

**GO'yu yükleyelim (Tek komutta)**

```
ver="1.18.1" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

## Node'u hazırlayalım

UPD - Blok sayısı 70,500'e gelince güncelleyin (Halihazırda bir node çalıştırıyorsanız)

```
systemctl stop strided
cd $HOME && rm -rf stride
git clone https://github.com/Stride-Labs/stride && cd stride
git checkout 90859d68d39b53333c303809ee0765add2e59dab
go build -mod=readonly -trimpath -o $HOME/go/bin ./...
systemctl restart strided && journalctl -u strided -f -o cat
```

:heavy_exclamation_mark:ÖNEMLİ - Aşağıdaki komutlarda <> içindekileri kendi bilgilerinizle değiştirin ve <> işaretini kaldırın:heavy_exclamation_mark:

```
git clone https://github.com/Stride-Labs/stride && cd stride
git checkout cf4e7f2d4ffe2002997428dbb1c530614b85df1b
mkdir -p $HOME/go/bin
go build -mod=readonly -trimpath -o $HOME/go/bin ./...
strided version --long | head
```
Versiyonu kontrol edelim
```
strided version --long | head
```
Gerekli ayar dosyalarını oluşturmak için bir node kuralım
```
strided init <name_moniker> --chain-id STRIDE-TESTNET-4
```
Genesis dosyasını indirelim
```
wget -O $HOME/.stride/config/genesis.json "https://raw.githubusercontent.com/Stride-Labs/testnet/main/poolparty/genesis.json"
```
Genesis dosyası için versiyon kontrolü yapalım
```
sha256sum ~/.stride/config/genesis.json
```
Başlangıç aşamasında Validator bloklarının durumunu kontrol edelim
```
cd && cat .stride/data/priv_validator_state.json
```
`{
  "height": "0",
  "round": 0,
  "step": 0
}`

Çıkan değer 0 değilse aşağıdaki komutu çalıştırın. 0 ise bu adımı geçin
```
strided tendermint unsafe-reset-all --home $HOME/.stride
```
addrbook dosyasını indirelim
```
wget -O $HOME/.stride/config/addrbook.json "https://raw.githubusercontent.com/doxe1/testnet-manuals-doxe/Stride/addrbook.json"
```
## Node ayarlarını yapalım
client.toml içindeki chain-id kısmını değiştirelim ki her seferinde yazmak zorunda kalmayalım
```
strided config chain-id STRIDE-TESTNET-4
```
Eğer gerekliyse `client.toml` içindeki keyring-backend'i değiştirelim
```
strided config keyring-backend os
```
`app.toml` içindeki münümum gaz ücretini ayarlayalım
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025ustrd\"/;" ~/.stride/config/app.toml
```
`config.toml` içine seed'leri/peer'leri ekleyelim
```
external_address=$(wget -qO- eth0.me)
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.stride/config/config.toml
```
```
peers="b11187784240586475422b132a3dcbc970a996dd@stride-node1.poolparty.stridenet.co:26656,2312417b613c44164bf167cb232795e7d8815be7@65.108.76.44:11523,84ff28824a911409e2c24f2f5ede87ae1b859b5f@5.189.178.222:46656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.stride/config/config.toml
```
```
seeds="baee9ccc2496c2e3bebd54d369c3b788f9473be9@seedv1.poolparty.stridenet.co:26656"
```
`config.toml` içindeki kalıcı peer'ler hariç gelen ve giden peer'leri çoğaltalım
```
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 100/g' $HOME/.stride/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 100/g' $HOME/.stride/config/config.toml
```
**(Opsiyonel)** Prnunning ayarlama `app.toml`
```
pruning="custom" && \
pruning_keep_recent="100" && \
pruning_keep_every="0" && \
pruning_interval="50" && \
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.stride/config/app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.stride/config/app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.stride/config/app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.stride/config/app.toml
```
**(Opsiyonel)** `config.toml` içindeki indexlemeyi kapatma
```
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.stride/config/config.toml
```
**(Opsiyonel)** `app.toml` içinde snapshot açma/kapatma
```
snapshot_interval=1000 && \
sed -i.bak -e "s/^snapshot-interval *=.*/snapshot-interval = \"$snapshot_interval\"/" ~/.stride/config/app.toml
```
Varsayılan olarak snapshot'lar devredışıdır `snapshot-interval=0`
## Servis dosyası oluşturalım
```
sudo tee /etc/systemd/system/strided.service > /dev/null <<EOF
[Unit]
Description=strided
After=network-online.target

[Service]
User=$USER
ExecStart=$(which strided) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload && \
sudo systemctl enable strided && \
sudo systemctl restart strided && sudo journalctl -u strided -f -o cat
```
:heavy_exclamation_mark:Node başladıktan uzun süre sonra hala peer bulamadıysa yeni peer'lar ekleyin ya da discord kanalına addrbook.json dosyasını sorun. Yardımvı olacaklardır. :heavy_exclamation_mark:

Node durdur, addrbook.json dosyasını ayarla, Node resetle
```
sudo systemctl stop strided
rm $HOME/.stride/config/addrbook.json
strided tendermint unsafe-reset-all --home $HOME/.stride
```
Node'u yeniden başlatalım
```
sudo systemctl restart strided && journalctl -u strided -f -o cat
```
Yeni bir cüzdan oluşturun ya da eski cüzdanınızın mnemonic kelimeleri ile eski cüzdanınızı kurtarın

:heavy_exclamation_mark:Mnemonic kelimelerinizi güvenli bir yerde saklayın. Çünkü kendi cüzdanınızın güvenliğinden siz sorumlusunuz:heavy_exclamation_mark:

Yeni bir cüzdan oluşturma
```
strided keys add <name_wallet> --keyring-backend os
```
Eski cüzdanı kurtarma
```
strided keys add <name_wallet> --recover --keyring-backend os
```
Ledger cüzdanı bağlama
```
strided keys add <name_wallet> --ledger 
```
Vlidator oluşturalım

:heavy_exclamation_mark:`priv_validator_key.json` dosyanızı yedeklemeyi unutmayın 
```
strided tx staking create-validator \
--chain-id STRIDE-TESTNET-4 \
--commission-rate 0.05 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.1 \
--min-self-delegation "1000000" \
--amount 1000000ustrd \
--pubkey $(strided tendermint show-validator) \
--moniker "<name_moniker>" \
--from <name_wallet> \
--fees 555ustrd
```
## Useful Commands

Sync sorgulama (Explorer'den güncel bloğa bakın eşleştiyse ve false gördüyseniz validator oluşturabilirsiniz)
```
strided status 2>&1 | jq ."SyncInfo"."latest_block_height"
```
Loglar akıyorr
```
sudo journalctl -u strided -f -o cat
sudo journalctl --lines=100 --follow --unit strided
```
Node durumu sorgulama
```
curl localhost:26657/status
```
Cüzdan bakiyesi sorgulama
```
strided q bank balances <address>
```
Ödülleri toplama komutu
```
strided tx distribution withdraw-rewards <valoper_address> --from <name_wallet> --fees 500ustrd --commission -y
```
Token'ları kendi cüzdanına delege etme (Örneğin: bir coin, bir sent)
```
strided tx staking delegate <valoper_address> 1000000ustrd --from <name_wallet> --fees 500ustrd -y
```
Başka bir cüzdana bakiye yollama
```
strided tx bank send <name_wallet> <address> 1000000ustrd --fees 500ustrd -y
```
Hapisten çıkma
```
strided tx slashing unjail --from <name_wallet> --fees 500ustrd -y
```
Cüzdan listesini gösterme komutu
```
strided keys list
```
Bir oylama için oy verme 
```
strided tx gov vote 1 yes --from <name_wallet> --fees 555ustrd
```
Node silme
```
sudo systemctl stop strided && \
sudo systemctl disable strided && \
rm /etc/systemd/system/strided.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf .stride stride && \
rm -rf $(which strided)
```
