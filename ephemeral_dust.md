# Testing ephemeral dust on Bitcoin 29

The following instructions are meant to be followed after the [29.0 Relase Testing Guide](https://github.com/bitcoin-core/bitcoin-devwiki/wiki/29.0-Release-Candidate-Testing-Guide), you can refer to this guide for the initial environment setup.

### Relay transaction packages containing ephemeral dust

An ephemeral dust output is a small output (below the dust treshold) that is created in a v3 zero-fee transaction and spent in a subsequent v3 transaction. The two transactions must be part of the same package. The output is ephemeral because it is both created and spent simultaneously. ([30239](https://github.com/bitcoin/bitcoin/pull/30239))

#### Steps to Test

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
  # ... in total 101 hashes are displayed ...
]
```
We now have one regtest spendable UTXO.
```bash
bcli29 -regtest listunspent
```
```bash
[
  {
    "txid": "497de4de4a54ac38bb9142bf0b1db743c5a972b71ec0b40adcbb42fca09dd225",
    "vout": 0,
    "address": "bcrt1qptldwfem24ueajmpmxny65het0f067ah5jvuhx",
    "label": "",
    "scriptPubKey": "00140afed7273b55799ecb61d9a64d52f95bd2fd7bb7",
    "amount": 50.00000000,
    "confirmations": 101,
    "spendable": true,
    "solvable": true,
    "desc": "wpkh([d072d21e/84h/1h/0h/0/0]02d9886aab1b63baf6813d435693f2f8be99252a8950f0796f7ad883901ec20ad8)#h6yht0sm",
    "parent_descs": [
      "wpkh(tpubD6NzVbkrYhZ4WgLPYQCrGAdp7Xp6sFL4w846tixR6aNFH872oZLcb41i9y37J2GuKydn4KvxuPYAexhtuMrhYhYb6jGfLVC61NKEQuRPpu3/84h/1h/0h/0/*)#wn9w4n2v"
    ],
    "safe": true
  }
]
```

### Create ephemeral dust

We will send the ephemeral dust output to a new address, so we need to get one first:
```bash
bcli29 -regtest getnewaddress
```
```bash
bcrt1q7wyyxrlrx0vxuqvt0k6x8x4r9uurezslsm6vjw
```
We now create a zero-fee transaction spending our UTXO. The transaction has one zero-amount output (the ephemeral dust output) and another output that sends back the rest of the funds.
```bash
bcli29 -regtest createrawtransaction '[{"txid":"497de4de4a54ac38bb9142bf0b1db743c5a972b71ec0b40adcbb42fca09dd225","vout":0}]' \
    '[{"bcrt1q7wyyxrlrx0vxuqvt0k6x8x4r9uurezslsm6vjw":0},{"bcrt1qptldwfem24ueajmpmxny65het0f067ah5jvuhx":50}]'
```
```bash
020000000125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```
We copy the result of the previous command in a text editor and change the version number at the beginning of the transaction, this is required because only v3 transactions support the new relay policy.
We get this result:
```
030000000125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```

### Destroy ephemeral dust

We decode the previously created transaction to get the transaction id and the scriptPubKey hex value, that are required in the next steps.
```bash
bcli29 -regtest decoderawtransaction 030000000125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```
```bash
{
  "txid": "e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612",
  "hash": "e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612",
  "version": 3,
  "size": 113,
  "vsize": 113,
  "weight": 452,
  "locktime": 0,
  "vin": [
    {
      "txid": "497de4de4a54ac38bb9142bf0b1db743c5a972b71ec0b40adcbb42fca09dd225",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00000000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 f388430fe333d86e018b7db4639aa32f383c8a1f",
        "desc": "addr(bcrt1q7wyyxrlrx0vxuqvt0k6x8x4r9uurezslsm6vjw)#uq067pg3",
        "hex": "0014f388430fe333d86e018b7db4639aa32f383c8a1f",
        "address": "bcrt1q7wyyxrlrx0vxuqvt0k6x8x4r9uurezslsm6vjw",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 50.00000000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 0afed7273b55799ecb61d9a64d52f95bd2fd7bb7",
        "desc": "addr(bcrt1qptldwfem24ueajmpmxny65het0f067ah5jvuhx)#4pgn6tg7",
        "hex": "00140afed7273b55799ecb61d9a64d52f95bd2fd7bb7",
        "address": "bcrt1qptldwfem24ueajmpmxny65het0f067ah5jvuhx",
        "type": "witness_v0_keyhash"
      }
    }
  ]
}
```
We now want to spend this transaction outputs. The child transaction must also spend the ephemeral dust to be accepted. We spend both outputs of the previous transaction and we add a fee that will pay for the entire package.
```bash
bcli29 -regtest createrawtransaction '[{"txid":"e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612","vout":0},{"txid":"e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612","vout":1}]' \
  '[{"bcrt1qptldwfem24ueajmpmxny65het0f067ah5jvuhx":49.995}]'
```
```bash
020000000212f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10000000000fdffffff12f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10100000000fdffffff01e050fe29010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```
Also in this case, we need to convert to a v3 transaction:
```
030000000212f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10000000000fdffffff12f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10100000000fdffffff01e050fe29010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```
### Signing
We sign the first transaction:
```bash
bcli29 -regtest signrawtransactionwithwallet 030000000125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000
```
```bash
{
  "hex": "0300000000010125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb7024730440220374145de40a11bcbd026ea561f875ec860215dd7bd25398016176704e337f17102201824e4eb302ae69d90c3543499e827fe0c7fac6720674eb397bbb9dbc144d3a1012102d9886aab1b63baf6813d435693f2f8be99252a8950f0796f7ad883901ec20ad800000000",
  "complete": true
}
```
And we sign the second transaction, by also providing the txid, vout, scriptPubKey and amount of the outputs of the first transaction.
```bash
bcli29 -regtest signrawtransactionwithwallet 030000000212f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10000000000fdffffff12f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10100000000fdffffff01e050fe29010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb700000000 \
  '[{"txid":"e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612","vout":0,"scriptPubKey":"0014f388430fe333d86e018b7db4639aa32f383c8a1f","amount":0},{"txid":"e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612","vout":1,"scriptPubKey":"00140afed7273b55799ecb61d9a64d52f95bd2fd7bb7","amount":50}]'
```
```
{
  "hex": "0300000000010212f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10000000000fdffffff12f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10100000000fdffffff01e050fe29010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb70247304402204ea136829cd08c80b01389c33eff37ca6e0b3e6a11b20fbf63ddb6dfc29465a8022010d36e685dd4ae0a1c5a439bbccb95ee1a2e45b612c42bd1f1e0da5e46fa759d0121025411fadcdf6e41703eeba9b5e507d0de43863e1e2cfb1ef5dcdd8ed3e62bbf430247304402201d9295f0f58708e7286eaa5f0e078febd2c9f3da41bfd7c9e58a10d1d4ff75e9022056246286429756aca4b3f7456f68345fafd199a1b4c7c5480c74f8cfd9fb18c4012102d9886aab1b63baf6813d435693f2f8be99252a8950f0796f7ad883901ec20ad800000000",
  "complete": true
}
```
We now try to submit the package containing the two signed transactions to the mempool:
```bash
bcli29 -regtest submitpackage '["0300000000010125d29da0fc42bbdc0ab4c01eb772a9c543b71d0bbf4291bb38ac544adee47d490000000000fdffffff020000000000000000160014f388430fe333d86e018b7db4639aa32f383c8a1f00f2052a010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb7024730440220374145de40a11bcbd026ea561f875ec860215dd7bd25398016176704e337f17102201824e4eb302ae69d90c3543499e827fe0c7fac6720674eb397bbb9dbc144d3a1012102d9886aab1b63baf6813d435693f2f8be99252a8950f0796f7ad883901ec20ad800000000","0300000000010212f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10000000000fdffffff12f6c331ce6cddbd136504b4cbadb1488a131d657daecef352fec26c8c9b3fe10100000000fdffffff01e050fe29010000001600140afed7273b55799ecb61d9a64d52f95bd2fd7bb70247304402204ea136829cd08c80b01389c33eff37ca6e0b3e6a11b20fbf63ddb6dfc29465a8022010d36e685dd4ae0a1c5a439bbccb95ee1a2e45b612c42bd1f1e0da5e46fa759d0121025411fadcdf6e41703eeba9b5e507d0de43863e1e2cfb1ef5dcdd8ed3e62bbf430247304402201d9295f0f58708e7286eaa5f0e078febd2c9f3da41bfd7c9e58a10d1d4ff75e9022056246286429756aca4b3f7456f68345fafd199a1b4c7c5480c74f8cfd9fb18c4012102d9886aab1b63baf6813d435693f2f8be99252a8950f0796f7ad883901ec20ad800000000"]'
```
```bash
{
  "package_msg": "success",
  "tx-results": {
    "a57fc569502539603d4ac2b675b35c51f63a62d0cc028434040cff199236ecdc": {
      "txid": "e13f9b8c6cc2fe52f3ceae7d651d138a48b1adcbb4046513bddd6cce31c3f612",
      "vsize": 141,
      "fees": {
        "base": 0.00000000,
        "effective-feerate": 0.01572327,
        "effective-includes": [
          "a57fc569502539603d4ac2b675b35c51f63a62d0cc028434040cff199236ecdc",
          "770b84163ed471e335ec188b7000b7d5dd774b7927d52c012b5a63bdabf2c7e9"
        ]
      }
    },
    "770b84163ed471e335ec188b7000b7d5dd774b7927d52c012b5a63bdabf2c7e9": {
      "txid": "d9b1e72acd02f3f63b82f4dc37603e3840b5f107271f2a58b77d2e42de5c6fde",
      "vsize": 177,
      "fees": {
        "base": 0.00500000,
        "effective-feerate": 0.01572327,
        "effective-includes": [
          "a57fc569502539603d4ac2b675b35c51f63a62d0cc028434040cff199236ecdc",
          "770b84163ed471e335ec188b7000b7d5dd774b7927d52c012b5a63bdabf2c7e9"
        ]
      }
    }
  },
  "replaced-transactions": [
  ]
}
```
The same command on Bitcoin 28.1 would have failed:
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

#### Ideas for other tests:
- try with v2 transansactions (one of them is enough): the package should be rejected
- add other non-zero outputs to the first transaction: the package should be accepted
- add other dust outputs to the first transaction: the package should be rejected
- in the second transaction, do not spend the ephemeral dust: the package should be rejected
- add a fee to the first transaction, the package should be rejected
- use a Pay-To-Anchor script in the dust output: the package should be accepted
