# C4 Block explorer
Bitcoin/Altcoin block explorer (unfinished)

An attempt to write a blockexplorer with an insight explorer compatible API.<br>
Unfinished explorer and needs a lot of refactoring since it was my first attempt to write a Golang application.<br>


![alt text](https://github.com/romanornr/blockexplorer/blob/dev/screenshots/1.png?raw=true)
<br></br>
![alt text](https://github.com/romanornr/blockexplorer/blob/dev/screenshots/2.png?raw=true)
<br></br>
![alt text](https://github.com/romanornr/blockexplorer/blob/dev/screenshots/3.png?raw=true)

##### Requirements

- **Go 1.10 or 1.11**
* Linux
* MongoDB
* dep https://github.com/golang/dep
* btc or altcoin wallet with RPC ports etc configured.

example config: ````cat ~/.viacoin/viacoin.conf````
```
rpcuser=via
rpcpassword=via
rpcallowip=127.0.0.1
listen=1
server=1
daemon=1
txindex=1
whitelist=127.0.0.1
zmqpubhashblock=tcp://127.0.0.1:28332
zmqpubrawblock=tcp://127.0.0.1:28332
zmqrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28332
```

and in config/app.yml
```yaml
server:
  ip: 127.0.0.1
  port: 8000

coin:
  name: Viacoin
  symbol: via
  algo: scrypt
  blocktime: 24

rpc:
  ip: 127.0.0.1
  username: via
  password: via
  port: 5222

zmq:
  endpoint: tcp://127.0.0.1:28332
```


  Installation instructions can be found here: https://golang.org/doc/install.
  It is recommended to add `$GOPATH/bin` to your `PATH` at this point.


#### setup
```bash
cd ~/go/src/gitlab.com/
```
```bash
git clone git@gitlab.com:romanornr/blockexplorer.git
```

```bash
dep ensure
```

##### Build database
To create the database for the block explorer use
```bash
go run cmd/build.go
```
or build the binary with
```bash
go build cmd/build.go
./build
```

##### Run webserver
```bash
go run main.go
```

or build a binary with
```bash
go build main.go
```

Make sure the core wallet for the asset is running with the right RPC auth details<br>
and make sure mongodb is running before you start.

##### TODO
- [ ] Solving Reorgs (priority)
- [ ] Complete frontend with Vuejs
- [ ] Adress validation
- [ ] Subsidy calculation for different coins instead of only viacoin
- [ ] API endpoint block raw
- [ ] Mempool unconfirmed transactions
- [ ] API block summaries
- [ ] API address properties
- [ ] API query multiple addresses
- [ ] Transaction broadcasting
- [ ] Access mogodbd from remote host (assumes localhost now)
- [ ] Support Postgresql too instead of Mongodb only

#### API 
```
[
  {
    "address":"mo9ncXisMeAoXwqcV5EWuyncbmCcQN4rVs",
    "txid":"d5f8a96faccf79d4c087fa217627bb1120e83f8ea1a7d84b1de4277ead9bbac1",
    "vout":0,
    "scriptPubKey":"76a91453c0307d6851aa0ce7825ba883c6bd9ad242b48688ac",
    "amount":0.000006,
    "satoshis":600,
    "confirmations":0,
    "ts":1461349425
  },
  {
    "address": "mo9ncXisMeAoXwqcV5EWuyncbmCcQN4rVs",
    "txid": "bc9df3b92120feaee4edc80963d8ed59d6a78ea0defef3ec3cb374f2015bfc6e",
    "vout": 1,
    "scriptPubKey": "76a91453c0307d6851aa0ce7825ba883c6bd9ad242b48688ac",
    "amount": 0.12345678,
    "satoshis: 12345678,
    "confirmations": 1,
    "height": 300001
  }
]
```
The `timestamp` property will only be set for unconfirmed transactions and `height` can be used for determining block order. The `confirmationsFromCache` is nolonger set or necessary, confirmation count is only cached for the time between blocks.

There is a new `GET` endpoint or raw blocks at `/rawblock/<blockHash>`:

Response format:
```
{
  "rawblock": "blockhexstring..."
}
```

There are a few changes to the `GET` endpoint for `/addr/[:address]`:

- The list of txids in an address summary does not include orphaned transactions
- The txids will be sorted in block order
- The list of txids will be limited at 1000 txids
- There are two new query options "from" and "to" for pagination of the txids (e.g. `/addr/[:address]?from=1000&to=2000`)

Some additional general notes:
- The transaction history for an address will be sorted in block order
- The response for the `/sync` endpoint does not include `startTs` and `endTs` as the sync is no longer relevant as indexes are built in bitcoind.
- The endpoint for `/peer` is no longer relevant connection to bitcoind is via ZMQ.
- `/tx` endpoint results will now include block height, and spentTx related fields will be set to `null` if unspent.
- `/block` endpoint results does not include `confirmations` and will include `poolInfo`.

## API HTTP Endpoints

### Block
```
  /api/block/[:hash]
  /api/block/00000000a967199a2fad0877433c93df785a8d8ce062e5f9b451cd1397bdbf62
```

### Block Index
Get block hash by height
```
  /api/block-index/[:height]
  /api/block-index/0
```
This would return:
```
{
  "blockHash":"000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f"
}
```
which is the hash of the Genesis block (0 height)


### Raw Block
```
  /api/rawblock/[:blockHash]
  /api/rawblock/[:blockHeight]
```

This would return:
```
{
  "rawblock":"blockhexstring..."
}
```

### Block Summaries

Get block summaries by date:
```
  /api/blocks?limit=3&blockDate=2016-04-22
```

Example response:
```
{
  "blocks": [
    {
      "height": 408495,
      "size": 989237,
      "hash": "00000000000000000108a1f4d4db839702d72f16561b1154600a26c453ecb378",
      "time": 1461360083,
      "txlength": 1695,
      "poolInfo": {
        "poolName": "BTCC Pool",
        "url": "https://pool.btcc.com/"
      }
    }
  ],
  "length": 1,
  "pagination": {
    "next": "2016-04-23",
    "prev": "2016-04-21",
    "currentTs": 1461369599,
    "current": "2016-04-22",
    "isToday": true,
    "more": true,
    "moreTs": 1461369600
  }
}
```

### Transaction
```
  /api/tx/[:txid]
  /api/tx/525de308971eabd941b139f46c7198b5af9479325c2395db7f2fb5ae8562556c
  /api/rawtx/[:rawid]
  /api/rawtx/525de308971eabd941b139f46c7198b5af9479325c2395db7f2fb5ae8562556c
```

### Address
```
  /api/addr/[:addr]
  /api/addr/[:addr][?noTxList=1][&from=&to=]
  /api/addr/mmvP3mTe53qxHdPqXEvdu8WdC7GfQ2vmx5?noTxList=1
  /api/addr/mmvP3mTe53qxHdPqXEvdu8WdC7GfQ2vmx5?from=1000&to=2000
```

### Address Properties
```
  /api/addr/[:addr]/balance
  /api/addr/[:addr]/totalReceived
  /api/addr/[:addr]/totalSent
  /api/addr/[:addr]/unconfirmedBalance
```
The response contains the value in Satoshis.

### Unspent Outputs (TODO)
```
  /api/addr/[:addr]/utxo
```
Sample return:
```
[
  {
    "address":"mo9ncXisMeAoXwqcV5EWuyncbmCcQN4rVs",
    "txid":"d5f8a96faccf79d4c087fa217627bb1120e83f8ea1a7d84b1de4277ead9bbac1",
    "vout":0,
    "scriptPubKey":"76a91453c0307d6851aa0ce7825ba883c6bd9ad242b48688ac",
    "amount":0.000006,
    "satoshis":600,
    "confirmations":0,
    "ts":1461349425
  },
  {
    "address": "mo9ncXisMeAoXwqcV5EWuyncbmCcQN4rVs",
    "txid": "bc9df3b92120feaee4edc80963d8ed59d6a78ea0defef3ec3cb374f2015bfc6e",
    "vout": 1,
    "scriptPubKey": "76a91453c0307d6851aa0ce7825ba883c6bd9ad242b48688ac",
    "amount": 0.12345678,
    "satoshis: 12345678,
    "confirmations": 1,
    "height": 300001
  }
]
```

### Unspent Outputs for Multiple Addresses (TODO)
GET method:
```
  /api/addrs/[:addrs]/utxo
  /api/addrs/2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f/utxo
```

POST method:
```
  /api/addrs/utxo
```

POST params:
```
addrs: 2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f
```

### Transactions by Block
```
  /api/txs/?block=HASH
  /api/txs/?block=00000000fa6cf7367e50ad14eb0ca4737131f256fc4c5841fd3c3f140140e6b6
```
### Transactions by Address
```
  /api/txs/?address=ADDR
  /api/txs/?address=mmhmMNfBiZZ37g1tgg2t8DDbNoEdqKVxAL
```

### Transactions for Multiple Addresses (TODO)
GET method:
```
  /api/addrs/[:addrs]/txs[?from=&to=]
  /api/addrs/2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f/txs?from=0&to=20
```

POST method:
```
  /api/addrs/txs
```

POST params:
```
addrs: 2NF2baYuJAkCKo5onjUKEPdARQkZ6SYyKd5,2NAre8sX2povnjy4aeiHKeEh97Qhn97tB1f
from (optional): 0
to (optional): 20
noAsm (optional): 1 (will omit script asm from results)
noScriptSig (optional): 1 (will omit the scriptSig from all inputs)
noSpent (option): 1 (will omit spent information per output)
```

Sample output:
```
{ totalItems: 100,
  from: 0,
  to: 20,
  items:
    [ { txid: '3e81723d069b12983b2ef694c9782d32fca26cc978de744acbc32c3d3496e915',
       version: 1,
       locktime: 0,
       vin: [Object],
       vout: [Object],
       blockhash: '00000000011a135e5277f5493c52c66829792392632b8b65429cf07ad3c47a6c',
       confirmations: 109367,
       time: 1393659685,
       blocktime: 1393659685,
       valueOut: 0.3453,
       size: 225,
       firstSeenTs: undefined,
       valueIn: 0.3454,
       fees: 0.0001 },
      { ... },
      { ... },
      ...
      { ... }
    ]
 }
```

Note: if pagination params are not specified, the result is an array of transactions.

### Transaction Broadcasting (TODO)
POST method:
```
  /api/tx/send
```
POST params:
```
  rawtx: "signed transaction as hex string"

  eg

  rawtx: 01000000017b1eabe0209b1fe794124575ef807057c77ada2138ae4fa8d6c4de0398a14f3f00000000494830450221008949f0cb400094ad2b5eb399d59d01c14d73d8fe6e96df1a7150deb388ab8935022079656090d7f6bac4c9a94e0aad311a4268e082a725f8aeae0573fb12ff866a5f01ffffffff01f0ca052a010000001976a914cbc20a7664f2f69e5355aa427045bc15e7c6c77288ac00000000

```
POST response:
```
  {
      txid: [:txid]
  }

  eg

  {
      txid: "c7736a0a0046d5a8cc61c8c3c2821d4d7517f5de2bc66a966011aaa79965ffba"
  }
```

### Historic Blockchain Data Sync Status (TODO)
```
  /api/sync
```

### Live Network P2P Data Sync Status (TODO)
```
  /api/peer
```

### Status of the Bitcoin Network (TODO)
```
  /api/status?q=xxx
```

Where "xxx" can be:

 * getInfo
 * getDifficulty
 * getBestBlockHash
 * getLastBlockHash
