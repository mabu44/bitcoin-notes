# Testing ephemeral dust on Bitcoin 29

The following instructions are meant to be followed after the [29.0 Relase Testing Guide](https://github.com/bitcoin-core/bitcoin-devwiki/wiki/29.0-Release-Candidate-Testing-Guide), you can refer to this guide for the initial environment setup.

### Relay transaction packages containing ephemeral dust

An ephemeral dust output is a small output (below the dust treshold) that is created in a v3 zero-fee transaction and spent in a subsequent v3 transaction. The two transactions must be part of the same package. The output is ephemeral because it is both created and spent simultaneously. ([30239](https://github.com/bitcoin/bitcoin/pull/30239))

#### Get some regtest coins

Change the bitcoin.conf file to run on regtest.
```bash
echo "regtest=1" > $DATA_DIR_29/bitcoin.conf
```
Start the bitcoind session
```bash
bitcoind29 daemon
```
Create a wallet
```bash
bcli29 -regtest createwallet ephemeral_dust
```
```bash
{
  "name": "ephemeral_dust"
}
```
Mine 101 blocks.
```bash
bcli29 -regtest generatetoaddress 101 $(bcli29 -regtest getnewaddress)
```
```bash
[
  # ... 101 hashes are displayed ...
]
```
We now have one regtest spendable UTXO and we can get the id of the transaction that generated it.
```bash
coinbase_txid=$(bcli29 -regtest listunspent | jq -r ".[0].txid")
echo $coinbase_txid
```
```bash
f20d0afe42fbda3e53d6b07dc3f0358af275685471f053c2fd46d4fa023123f3
```

#### Create an ephemeral dust output

We need two new addresses: one for the dust output and one to which the coins will be sent.
```bash
dust_address=$(bcli29 -regtest getnewaddress)
coins_address=$(bcli29 -regtest getnewaddress)
echo $dust_address
echo $coins_address
```
```bash
bcrt1q8laca583j8wfeu8pcyjej2dtaajrcrg7urvra8
bcrt1q8e5vlev9tvphtxpz7y2tat239u5j6c9ly30w2p
```
We now create a zero-fee transaction spending our UTXO. The transaction has one zero-amount output (the ephemeral dust output) and a change output.
```bash
tx1_v2=$(bcli29 -regtest createrawtransaction "[{\"txid\":\"$coinbase_txid\",\"vout\":0}]" \
    "[{\"$dust_address\":0},{\"$coins_address\":50}]")
echo $tx1_v2
```
```bash
0200000001f3233102fad446fdc253f071546875f28a35f0c37db0d6533edafb42fe0a0df20000000000fdffffff0200000000000000001600143ffb8ed0f191dc9cf0e1c1259929abef643c0d1e00f2052a010000001600143e68cfe5855b03759822f114bead512f292d60bf00000000
```
This is a v2 transaction but we need a v3 transaction. The format of the transaction remains the same, we only change the version bits by replacing the first two digits with "03".
```bash
tx1="03${tx1_v2:2}"
echo $tx1
```
```bash
0300000001f3233102fad446fdc253f071546875f28a35f0c37db0d6533edafb42fe0a0df20000000000fdffffff0200000000000000001600143ffb8ed0f191dc9cf0e1c1259929abef643c0d1e00f2052a010000001600143e68cfe5855b03759822f114bead512f292d60bf00000000
```

### Destroy ephemeral dust

We decode the previously created transaction to get the transaction id and the scriptPubKey hex value, that are required in the next steps.
```bash
tx1_decoded=$(bcli29 -regtest decoderawtransaction $tx1)
tx1_txid=$(echo $tx1_decoded | jq -r .txid)
tx1_script_dust=$(echo $tx1_decoded | jq -r ".vout[0].scriptPubKey.hex")
tx1_script_coins=$(echo $tx1_decoded | jq -r ".vout[1].scriptPubKey.hex")
echo $tx1_txid
echo $tx1_script_dust
echo $tx1_script_coins
```
```bash
33b70932a641432488cab448910c6ec2cec931daec7365a2c69de88427fafc22
00143ffb8ed0f191dc9cf0e1c1259929abef643c0d1e
00143e68cfe5855b03759822f114bead512f292d60bf
```
We now want to spend this transaction outputs. The child transaction must also spend the ephemeral dust to be accepted. We spend both outputs of the previous transaction and we add a fee that will pay for the entire package. Also in this case, we need to convert to a v3 transaction.
```bash
tx2_v2=$(bcli29 -regtest createrawtransaction "[{\"txid\":\"$tx1_txid\",\"vout\":0},{\"txid\":\"$tx1_txid\",\"vout\":1}]" "[{\"$coins_address\":49.995}]")
tx2="03${tx2_v2:2}"
echo $tx2
```
```bash
030000000222fcfa2784e89dc6a26573ecda31c9cec26e0c9148b4ca88244341a63209b7330000000000fdffffff22fcfa2784e89dc6a26573ecda31c9cec26e0c9148b4ca88244341a63209b7330100000000fdffffff01e050fe29010000001600143e68cfe5855b03759822f114bead512f292d60bf00000000
```

### Signing
We sign the first transaction:
```bash
tx1_signed=$(bcli29 -regtest signrawtransactionwithwallet $tx1 | jq -r .hex)
echo $tx1_signed
```
```bash
03000000000101f3233102fad446fdc253f071546875f28a35f0c37db0d6533edafb42fe0a0df20000000000fdffffff0200000000000000001600143ffb8ed0f191dc9cf0e1c1259929abef643c0d1e00f2052a010000001600143e68cfe5855b03759822f114bead512f292d60bf0247304402202f1e1a53fa11b593853f1535114243ff628ae71a8b769a5f0ffd5e0e8e576df402200cf126817b74e860fb11bc45147776f0757689dcd4ea760061e6504fa1fcb07c0121026efaada55665a9ce205c78764d5c42c802af1134eca662fbc118f5846aa95a8e00000000
```
And we sign the second transaction, by also providing the txid, vout, scriptPubKey and amount of the outputs of the first transaction.
```bash
tx2_signed=$(bcli29 -regtest signrawtransactionwithwallet $tx2 \
"[{\"txid\":\"$tx1_txid\",\"vout\":0,\"scriptPubKey\":\"$tx1_script_dust\",\"amount\":0},{\"txid\":\"$tx1_txid\",\"vout\":1,\"scriptPubKey\":\"$tx1_script_coins\",\"amount\":50}]" | jq -r .hex)
echo $tx2_signed
```
```bash
0300000000010222fcfa2784e89dc6a26573ecda31c9cec26e0c9148b4ca88244341a63209b7330000000000fdffffff22fcfa2784e89dc6a26573ecda31c9cec26e0c9148b4ca88244341a63209b7330100000000fdffffff01e050fe29010000001600143e68cfe5855b03759822f114bead512f292d60bf02473044022044a9a0c25309a7a09a4bb2731e3a4f31b4e22eb5af96b59e5395488b33126a8b02206d19e1e147193c2fad2189f19fa7f8802e6822892acad5935557d69e7683d63a012103ebd3fcba8eccdbfca60ae343d209b3132d9ef7d4f341f3214cc6fc44d8d216db024730440220019ff3cb31afcad56c6fee55194b2c35c50205ae2d3e4c6dfcc76cc737e067a60220691a56b43f6706be4af520c86e07aabe7257770022ac22d4b4378a6e5d1774c70121036ae7ee47b4a9575f2f5b4977650806b22389c98bde5a29f725af4435877f460600000000
```
We now try to submit the package containing the two signed transactions to the mempool:
```bash
bcli29 -regtest submitpackage "[\"$tx1_signed\",\"$tx2_signed\"]"
```
```bash
{
  "package_msg": "success",
  "tx-results": {
    "570cc77a1a3a981418c67feba39625b5b650f97aa106e0cfab14fbd1b6920143": {
      "txid": "33b70932a641432488cab448910c6ec2cec931daec7365a2c69de88427fafc22",
      "vsize": 141,
      "fees": {
        "base": 0.00000000,
        "effective-feerate": 0.01572327,
        "effective-includes": [
          "570cc77a1a3a981418c67feba39625b5b650f97aa106e0cfab14fbd1b6920143",
          "72893787a375f8d1fb8deed8b461855beb199463ba8e0d008e98d973198f01db"
        ]
      }
    },
    "72893787a375f8d1fb8deed8b461855beb199463ba8e0d008e98d973198f01db": {
      "txid": "8d328bede7afe34dc92a6f4bc9ff1398faa75883d0c85fe8cf04b6489bff901e",
      "vsize": 177,
      "fees": {
        "base": 0.00500000,
        "effective-feerate": 0.01572327,
        "effective-includes": [
          "570cc77a1a3a981418c67feba39625b5b650f97aa106e0cfab14fbd1b6920143",
          "72893787a375f8d1fb8deed8b461855beb199463ba8e0d008e98d973198f01db"
        ]
      }
    }
  },
  "replaced-transactions": [
  ]
}
```
The same command on Bitcoin 28.1 would have failed with a result similar to the following:
```bash
{
  "package_msg": "transaction failed",
  "tx-results": {
    "a57fc569502539603d4ac2b675b35c51f63a62d0cc028434040cff199236ecdc": {
      "txid": "e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612",
      "error": "dust"
    },
    "770b84163ed471e335ec188b7000b7d5dd774b7927d52c012b5a63bdabf2c7e9": {
      "txid": "d9b1e72acd02f3f63b82f4dc37603e3840b5f107271f2a58b77d2e42de5c6fde",
      "error": "bad-txns-inputs-missingorspent"
    }
  },
  "replaced-transactions": [
  ]
}
```

#### Ideas for further testing:
- try with v2 transansactions (one of them is enough): the package should be rejected
- add other non-dust outputs to the first transaction: the package should be accepted
- add other dust outputs to the first transaction: the package should be rejected
- in the second transaction, do not spend the ephemeral dust: the package should be rejected
- add a fee to the first transaction, the package should be rejected
- use a Pay-To-Anchor script in the dust output: the package should be accepted
