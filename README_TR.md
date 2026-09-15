# Worrell-Testnet-Node-Validator-Kurulum-Rehberi
Bu repo, Worrell Testnet ağı üzerinde tam düğüm (full node) kurma, hızlı senkronizasyon (State Sync) yapma ve validatör oluşturma adımlarını adım adım içermektedir.

# 📊 Ağ Bilgileri (Network Details)ParametreDeğer
Chain ID worrell-testnet-1
Binary Adıworrelld (Cosmos SDK v0.53.6)
Base Denom uworrell (1 WORRELL = 1,000,000 uworrell)
Min Gas Price 0.025uworrell

# 💻 Sistem Gereksinimleri
Kaynak Testnet(Önerilen)
İşletim Sistemi Ubuntu 22.04 LTS
CPU 2 vCPU
RAM 4 GB
Disk 100 GB SSD

# 🛠️ 1. Hazırlık ve Kurulum (Installation)Sunucunuzu güncelleyip gerekli temel araçları kurun:
```Bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git jq lz4 build-essential -y
```
Önceden derlenmiş (Prebuilt) dosyayı indirip kurun:
```Bash
curl -LO https://github.com/worrellchain/worrell/releases/download/v0.1.2/worrell_v0.1.2_linux_amd64.tar.gz
tar -xzf worrell_v0.1.2_linux_amd64.tar.gz
sudo mv worrelld /usr/local/bin/
```

# Sürüm kontrolü (cosmos_sdk_version: v0.53.6 olmalıdır)
```
worrelld version --long | head -5
```

# ⚙️ 2. Başlatma ve Yapılandırma (Init & Config)Kendi belirlediğiniz bir MONIKER (Node adı) ve WALLET (Cüzdan adı) girerek değişkenleri ayarlayın:
```Bash
export MONIKER="KENDI_NODE_ADINIZ"
export WALLET="KENDI_CUZDAN_ADINIZ"
```
Node'u başlatın ve resmi Genesis dosyasını indirin:
```Bash
# Node Init
worrelld init $MONIKER --chain-id worrell-testnet-1

# Resmi Genesis dosyasını indir
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-1/genesis.json > $HOME/.worrell/config/genesis.json

# Genesis Hash Kontrolü (Çıktı a81c... ile başlamalıdır)
sha256sum $HOME/.worrell/config/genesis.json
```
Peer ve Gas ayarlarını yapılandırın (Resmi ve ekstra topluluk peerleri eklenmiştir):

```Bash
# Persistent Peers ekleme
PEERS="bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656,40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656,e812f08760b18ed774369e899763735f80179f76@peer-worrell.grandvalleys.com:17656"
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"$PEERS\"|" $HOME/.worrell/config/config.toml

# Minimum Gas Price ayarı
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025uworrell\"|" $HOME/.worrell/config/app.toml
```

#  3. Hızlı Senkronizasyon (State Sync)Baştan senkronize olmak saatler sürebilir. Ağa hızlıca katılmak için ITRocket RPC'sini kullanarak State Sync yapıyoruz:

```Bash
# Node verilerini sıfırla (priv_validator_key saklanarak)
worrelld tendermint unsafe-reset-all --home $HOME/.worrell

SNAP_RPC="https://worrell-testnet-rpc.itrocket.net:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
TRUST_HEIGHT=$((LATEST_HEIGHT - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)

sed -i -E "s|^[[:space:]]*enable[[:space:]]*=.*|enable = true|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*rpc_servers[[:space:]]*=.*|rpc_servers = \"$SNAP_RPC,$SNAP_RPC\"|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*trust_height[[:space:]]*=.*|trust_height = $TRUST_HEIGHT|" $HOME/.worrell/config/config.toml
sed -i -E "s|^[[:space:]]*trust_hash[[:space:]]*=.*|trust_hash = \"$TRUST_HASH\"|" $HOME/.worrell/config/config.toml
```

# 🛡️ 4. Servis Oluşturma ve Başlatma (Systemd)Arka planda kesintisiz çalışması için Systemd servisi oluşturuyoruz:

```Bash
sudo tee /etc/systemd/system/worrelld.service > /dev/null <<EOF
[Unit]
Description=Worrell Testnet Node
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/worrelld start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# Servisi başlat
sudo systemctl daemon-reload
sudo systemctl enable --now worrelld
```

# 💰 5. Cüzdan ve Validatör İşlemleri
 Cüzdan Oluşturma ve Faucet

```Bash
# Yeni cüzdan oluştur (ÇIKAN GİZLİ KELİMELERİ MUTLAKA KAYDEDİN!)
worrelld keys add $WALLET

# Test token iste (Saatte bir 500 WORRELL alınabilir)
curl -X POST http://164.68.98.186:4500 \
  -H "Content-Type: application/json" \
  -d "{\"address\":\"$(worrelld keys show $WALLET -a)\"}"

# Bakiyeyi kontrol et
worrelld query bank balances $(worrelld keys show $WALLET -a)
```
ÖNEMLİ: Validatör kurmadan önce node'unuzun ağ ile tam senkronize olduğundan emin olun. catching_up değeri false olmalıdır. (Aşağıdaki takip komutları bölümünden bakabilirsiniz.)

 Validatör Oluşturma
 Bakiye hesabınıza geçtikten ve node senkronize olduktan sonra (Örn: 490 WORRELL kullanarak):
 
 ```Bash
cat <<EOF > $HOME/.worrell/validator.json
{
  "pubkey": $(worrelld tendermint show-validator),
  "amount": "490000000uworrell",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "Worrell Node Operator",
  "commission-rate": "0.05",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1000000"
}
EOF

worrelld tx staking create-validator $HOME/.worrell/validator.json \
  --from $WALLET \
  --chain-id worrell-testnet-1 \
  --gas auto \
  --gas-adjustment 1.5 \
  --gas-prices 0.025uworrell \
  -y
```

# 🔍 6. Operasyon, İnceleme ve Takip Komutları (Cheatsheet)
 İnceleme ve Log Takibi
 
```Bash
# Canlı log akışını izleme
sudo journalctl -u worrelld -f -o cat

# Senkronizasyon durumunu kontrol etme ("catching_up": false olmalıdır)
worrelld status 2>&1 | jq '.sync_info'

# Validatörünüzün missed blocks (kaçırılan bloklar) durumunu görme
worrelld query slashing signing-info $(worrelld tendermint show-address)
```
Validatör Yönetimi

```Bash
# Kendinize ekstra token delege etme (Örn: 100 WORRELL)
worrelld tx staking delegate $(worrelld keys show $WALLET --bech val -a) 100000000uworrell \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y

# Hapisten çıkma (Unjail)
worrelld tx slashing unjail \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y

# Komisyon oranını değiştirme (Günlük en fazla %1 artırılabilir. Örn: %6 için)
worrelld tx staking edit-validator \
  --commission-rate 0.06 \
  --from $WALLET --chain-id worrell-testnet-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.025uworrell -y
```

# 🔄 7. Güncelleme ve Silme (Update & Delete)

A. Binary Güncelleme (Yeni Sürüm Geldiğinde)
Yeni bir duyuru geldiğinde (Örn: v0.2.0) node'u durdurup binary'i değiştirmeniz gerekir:

```Bash
sudo systemctl stop worrelld
# Yeni sürümü indirip eskisinin üzerine yazın
curl -LO https://github.com/worrellchain/worrell/releases/download/vX.Y.Z/worrell_vX.Y.Z_linux_amd64.tar.gz
tar -xzf worrell_vX.Y.Z_linux_amd64.tar.gz
sudo mv worrelld /usr/local/bin/
# Servisi tekrar başlatın
sudo systemctl restart worrelld
sudo journalctl -u worrelld -f
```
B. Node'u Komple Silme (Dikkat!)
Eğer node'u sunucudan tamamen kaldırmak isterseniz (Gizli kelimelerinizi yedeklediğinizden emin olun):

```Bash
sudo systemctl stop worrelld
sudo systemctl disable worrelld
sudo rm /etc/systemd/system/worrelld.service
sudo systemctl daemon-reload
rm -rf $HOME/.worrell
sudo rm /usr/local/bin/worrelld
```
