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
