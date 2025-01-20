# Bitcoin transactions scripts

## General concepts
- Ouput script :lock: : is set when the UTXO is created, usually decoding the address provided by the receiver. It represents the condition to spend the funds. It is part of the outputs and it is stored in the UTXO set of all nodes.
- Input script :key: : the script that is used to unlock the UTXO depending on its output script. It must satisfy the conditions expressed in the output script. Is in the input so it is only stored on the blockchain but not in the UTXO set. 
- Witness :key: : it is used as an alternative to the input script in modern transactions, because its space is discounted in terms of fees.

An input script unlocks a given output script if the resulting program leaves a value on top of the stack that is not zero.

## P2PK
- Output script :lock: : `<PubKey> OP_CHECKSIG`
- Input script :key: : `<Sig>`

Resulting program :unlock: : `<Sig> <PubKey> OP_CHECKSIG`

## P2PKH
- Output script :lock: : `OP_DUP OP_HASH160 <PubKeyHash160> OP_EQUAL OP_CHECKSIG`
- Input script :key: : `<Sig> <PubKey>`

Resulting program :unlock: : `<Sig> <PubKey> OP_DUP OP_HASH160 <PubKeyHash160> OP_EQUAL OP_CHECKSIG`

## P2SH
- Output script :lock: : `OP_HASH160 <RedemScriptHash160> OP_EQUAL` (exactly as it is!)
- Input script :key: : `...data and signatures... <RedeemScript>`

Resulting program (if RedeemScript hash is correct) :unlock: : `...data and signatures... <RedeemScript>`

## P2WPKH
- Output script :lock: : `OP_0 <PubKeyHash160>` (exactly as it is!)
- Input script : empty
- Witness :key: : `<Sig> <PubKey>`

Resulting program :unlock: : `<Sig> <PubKey> OP_DUP OP_HASH160 <PubKeyHash160> OP_EQUAL OP_CHECKSIG` (most operators are implicit in the transaction but explicitly signed in \<Sig\>)

## P2WSH
- Output script :lock: : `OP_0 <RedeemScriptHash256>` (exactly as it is!)
- Input script : empty
- Witness :key: : \
`...data and signatures... <RedeemScript>`

Resulting program (if RedeemScript hash is correct) :unlock: : `...data and signatures... <RedeemScript>`

### P2WSH common example: m-of-n multisig
- Output script :lock: : `OP_0 <RedeemScriptHash256>` (exactly as it is!)
- Input script : empty
- Witness :key: : \
`OP_0 <Sig_1> ... <Sig_m>  <RedeemScript>`
- Redeem script: \
`<m> <PubKey_1> ... <PubKey_n> <n> OP_CHECKMULTISIG`

Resulting program (if RedeemScript hash is correct) :unlock: : \
`OP_0 <Sig_1> ... <Sig_m> <m> <PubKey_1> ... <PubKey_n> <n> OP_CHECKMULTISIG`
