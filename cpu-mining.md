# Mine bitcoin in a fully independent way

This draft provides some hints on running a Bitcoin mining full-node on a PC without GPUs, for **educational purposes**, as mining a block with a personal computer is **extremely** unlikely.

## 1. Run a Bitcoin client as a server

A fully synchronized Bitcoin full-node client should be running with, at least, the following options in *bitcoin.conf*:

```conf
rpcuser=[...]
rpcpassword=[...]
zmqpubhashblockhwm=tcp://127.0.0.1:28332
```

The rpcuser value and a secure password for the rpcpassword can be chosen.

The ZMQ protocol offers a more efficient alternative to polling, allowing the miner to receive notifications when a new block is propagated through the Bitcoin network. The port number used for the ZMQ protocol can be customized.

## 2. Run a solo pool locally
[CKPool](https://bitbucket.org/ckolivas/ckpool) can be easily compiled from sources, public pool is also well known.

The *ckpool.conf* file, located in the same folder as the ckpool executable, will contain settings similar to the following:

```json
{
"btcd" :  [
	{
		"url" : "localhost:8332",
		"auth" : "SEE_YOUR_RPCUSER_SETTING_ON_BITCOIN_CONF",
		"pass" : "SEE_YOUR_RPCPASSWORD_SETTING_ON_BITCOIN_CONF",
		"notify" : true
	}
],
"btcaddress" : "#(YOUR_BTC_ADDRESS)#",
"btcsig" : "/mined by ck/",
"blockpoll" : 100,
"donation" : 2.0,
"nonce1length" : 4,
"nonce2length" : 8,
"update_interval" : 30,
"version_mask" : "1fffe000",
"serverurl" : [
	"localhost:3334"
],
"mindiff" : 1,
"startdiff" : 1,
"maxdiff" : 0,
"zmqblock" : "tcp://127.0.0.1:28332",
"logdir" : "logs"
}
```

The "mindiff" and "startdiff" parameters have been set to 1, as the default value of 42 would be too demanding for the CPU. This change has only a cosmetic effect with no impact on the blockchain or (non-existing) potential profits.

The port of the "zmqblock" parameter has been set to the same value used in the configuration of the Bitcoin client.

The "donation" parameter sets the percentage of block reward donated to the CKPool software developer upon finding a block.

## 3. Point a miner to the pool

For CPU mining, [cpuminer](https://github.com/pooler/cpuminer/releases/tag/v2.5.1), can be used to connect to the local solo pool.
The command would be similar to this:

```bash
./minerd -o stratum+tcp://localhost:3334 -a sha256d -O user.cpuminer:x -D
```

Note that the port "3334" must be the same specified in the "serverurl" section of *ckpool.conf*.

## 4. Results

The Bitcoin client will periodically receive requests from the local solo pool to create block templates, and these events will be logged by the "bitcoind" process in a similar manner to the following:

```text
2025-05-25T15:44:37Z CreateNewBlock(): total size: 34065 block weight: 95838 txs: 136 fees: 597840 sigops 437
2025-05-25T15:46:37Z CreateNewBlock(): total size: 48034 block weight: 133819 txs: 187 fees: 727480 sigops 467
2025-05-25T15:48:37Z CreateNewBlock(): total size: 68125 block weight: 189307 txs: 260 fees: 954907 sigops 496
2025-05-25T15:50:37Z CreateNewBlock(): total size: 85453 block weight: 238426 txs: 331 fees: 1132346 sigops 518
2025-05-25T15:52:37Z CreateNewBlock(): total size: 105741 block weight: 293979 txs: 405 fees: 1389629 sigops 541
```

The CKPool software will log something along these lines:

```text
[2025-05-25 15:43:37.578] ckpool generator starting
[2025-05-25 15:43:37.578] ckpool stratifier starting
[2025-05-25 15:43:37.578] ckpool connector starting
[2025-05-25 15:43:37.579] ckpool connector ready
[2025-05-25 15:43:37.593] ckpool generator ready
[2025-05-25 15:43:37.593] Connected to bitcoind: localhost:8332
[2025-05-25 15:43:37.601] Loaded 1 users and 1 workers
[2025-05-25 15:43:37.601] ckpool stratifier ready
[2025-05-25 15:43:37.611] Mining from any incoming username to address [...]
[2025-05-25 15:43:37.611] Network diff set to 121754806822969
```

The next line is continuously updated and, after some minutes, takes on the following structure:

```text
[2025-05-25 15:53:49.608] - 12.8MH/s  0.0 SPS  1 users  1 workers  2 shares  0.0% diff
```

The cpuminer is also logging data relative to its job:

```text
[2025-05-25 15:43:53] Starting Stratum on stratum+tcp://localhost:3334
[2025-05-25 15:43:53] Binding thread 0 to cpu 0
[2025-05-25 15:43:53] 4 miner threads started, using 'sha256d' algorithm.
[2025-05-25 15:43:53] Binding thread 1 to cpu 1
[2025-05-25 15:43:53] Binding thread 2 to cpu 2
[2025-05-25 15:43:53] Binding thread 3 to cpu 3
[2025-05-25 15:43:53] Stratum session id: 68337ee4
[2025-05-25 15:43:53] Stratum difficulty set to 1
[2025-05-25 15:43:53] DEBUG: job_id='68337ee300000000' extranonce2=0000000000000000 ntime=68337f5d
[2025-05-25 15:43:53] Stratum requested work restart
[2025-05-25 15:43:53] thread 2: 2097152 hashes, 3133 khash/s
[2025-05-25 15:43:53] thread 3: 2097152 hashes, 3123 khash/s
[2025-05-25 15:43:53] thread 1: 2097152 hashes, 2981 khash/s
[2025-05-25 15:43:53] thread 0: 2097152 hashes, 2928 khash/s
```

and occasionally it will find a share, which is a block that meets the local solo pool's proof of work requirement, currently set to a difficulty of 1.

```text
[2025-05-25 15:45:48] DEBUG: hash <= target
Hash:   000000009856eab981c56dc335ef2f46c045204a2b3f67dda85a4d41ad306172
Target: 00000000ffff0000000000000000000000000000000000000000000000000000
[2025-05-25 15:45:48] thread 0: 132422944 hashes, 3149 khash/s
[2025-05-25 15:45:48] accepted: 1/1 (100.00%), 12981 khash/s (yay!!!)
```

This is sufficient proof of work for our local solo pool, but it falls short of the requirements for the Bitcoin network, which has a significantly higher difficulty target of around 121T. While it is theoretically possible to find a valid Bitcoin block in this way, the likelihood of this occurring is extremely low.

## 5. Mine on regtest to find a block

To test block creation, the above procedure can be run on [regtest](https://river.com/learn/terms/r/regtest/). However, please note that this is only possible if the current block height is at least 16, for [this reason](https://bitcoin.stackexchange.com/a/126220).

The first blocks can be created using the following commands:

```bash
bitcoin-cli createwallet 'miner'
temp_addr=$(bitcoin-cli getnewaddress 'generate' 'bech32')
bitcoin-cli generatetoaddress 16 $addr
```

On ckpool.conf, the new port for bitcoind is 18443 (default for regtest) and a regtest bitcoin address can be obtained with:

```bash
miner_addr=$(bitcoin-cli getnewaddress 'miner' 'bech32')
echo $miner_addr
```

When a share is produced by minerd, because the difficulty of the regtest network is 0, that share is also a valid block. CKPool will log something along those lines:

```log
[2025-05-26 17:25:33.025] Network diff set to 1.0
[2025-05-26 17:34:25.551] Possible block solve diff 1.257313 !
[2025-05-26 17:34:25.553] BLOCK ACCEPTED!
[2025-05-26 17:34:25.554] Solved and confirmed block 19 by user.cpuminer
[2025-05-26 17:34:25.554] User user:{"hashrate1m": "0", "hashrate5m": "0", "hashrate1hr": "0", "hashrate1d": "0", "hashrate7d": "0", "shares": 0, "authorised": 1748280340}
[2025-05-26 17:34:25.554] Worker user.cpuminer:{"hashrate1m": "0", "hashrate5m": "0", "hashrate1hr": "0", "hashrate1d": "0", "hashrate7d": "0"}
[2025-05-26 17:34:25.554] Block solved after 0 shares at 0.0% diff
[2025-05-26 17:34:25.554] Submitting possible block solve share diff 1.257313 !
```

And the Bitcoin client:

```log
2025-05-26T17:34:25Z Saw new header hash=00000000cb9b12ac8040a0f6bb451f348c7291f56c85a2ed4b2f5e70fae75142 height=19
2025-05-26T17:34:25Z UpdateTip: new best=00000000cb9b12ac8040a0f6bb451f348c7291f56c85a2ed4b2f5e70fae75142 height=19 version=0x20000000 log2_work=5.321928 tx=20 date='2025-05-26T17:32:33Z' progress=1.000000 cache=0.3MiB(19txo)
2025-05-26T17:34:25Z [miner] AddToWallet 427332b7fc187410561fabfa082b9fd42f87cd8d599a6d665023fabcd1709678  new Confirmed (block=00000000cb9b12ac8040a0f6bb451f348c7291f56c85a2ed4b2f5e70fae75142, height=19, index=0)
```

To double check that the new coins have been generated to the miner address, it is possible to run:

```bash
bitcoin-cli getaddressinfo $miner_addr
```

In the response the number of hashes in the "use_txids" field should be equal to the number of mined blocks.
