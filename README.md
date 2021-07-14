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
