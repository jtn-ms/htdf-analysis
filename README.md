# htdf-analysis
### DEBUG
```
[log]
hsd start --log_level "main:info,state:info,*:error"
LOG_LEVEL=trace hsd start

[wal]
$ cd github.com/orientwalt/tendermint
$ go run scripts/wal2json/main.go ~/.hsd/data/cs.wal/wal > ~/wal.json
```
### [CLevelDB](https://docs.tendermint.com/v0.33/introduction/install.html)
```
sudo apt-get update
sudo apt install build-essential

sudo apt-get install libsnappy-dev

wget https://github.com/google/leveldb/archive/v1.20.tar.gz && \
  tar -zxvf v1.20.tar.gz && \
  cd leveldb-1.20/ && \
  make && \
  sudo cp -r out-static/lib* out-shared/lib* /usr/local/lib/ && \
  cd include/ && \
  sudo cp -r leveldb /usr/local/include/ && \
  sudo ldconfig && \
  rm -f v1.20.tar.gz
  
# config/config.toml
db_backend = "cleveldb"

CGO_LDFLAGS="-lsnappy" make install TENDERMINT_BUILD_OPTIONS=cleveldb

CGO_LDFLAGS="-lsnappy" make build TENDERMINT_BUILD_OPTIONS=cleveldb
```
### PROMETHEUS
```
# config/config.toml
prometheus = true
```
### Reset to Genesis Block 
```
hsd unsafe-reset-all
```
### Replay Last Block
```
hsd start --replay-last-block
```
### [Upgrade](https://github.com/orientwalt/htdf/blob/v2.1.0/docs/upgrade.md)
```
[1. stop hsd]
[2. backup .hsd]
[3. update hsd]
[4. start hsd]

[5. submit proposal]
hscli tx gov submit-proposal htdf1sh8d3h0nn8t4e83crcql80wua7u3xtlfj5dej3 --gas-price=100  --switch-height=320 --description="second proposal"  --title="test1" --type="software_upgrade" --deposit="1000000000satoshi" --version="1"

[6. charge]
hscli tx send htdf1sh8d3h0nn8t4e83crcql80wua7u3xtlfj5dej3 htdf1xvktg68uwrtkml7m4yqul2sjm6fhtglm6y20eg 1000000000satoshi --gas-price=100
hscli tx send htdf1sh8d3h0nn8t4e83crcql80wua7u3xtlfj5dej3 htdf1p97p8vckpvkzx34se7eansm3rp223r3trmn63h 1000000000satoshi --gas-price=100
hscli tx send htdf1sh8d3h0nn8t4e83crcql80wua7u3xtlfj5dej3 htdf18efa8m8yudlqc765s9vkdrev5475jyqalqm5a0 1000000000satoshi --gas-price=100
hscli tx send htdf1sh8d3h0nn8t4e83crcql80wua7u3xtlfj5dej3 htdf1nuxf4amphaajuwg0ph3se6kmsda9cs6v0sja7r 1000000000satoshi --gas-price=100

[7. vote]
hscli tx gov vote  htdf1xvktg68uwrtkml7m4yqul2sjm6fhtglm6y20eg 2  yes --gas-price=100 
hscli tx gov vote  htdf1p97p8vckpvkzx34se7eansm3rp223r3trmn63h 2  yes --gas-price=100 
hscli tx gov vote  htdf18efa8m8yudlqc765s9vkdrev5475jyqalqm5a0 2  yes --gas-price=100 
hscli tx gov vote  htdf1nuxf4amphaajuwg0ph3se6kmsda9cs6v0sja7r 2  yes --gas-price=100 

[8. check]
hscli query staking params
unbonding, unslashing test
```
