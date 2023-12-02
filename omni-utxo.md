# Omnize-UTXO for Bitcoin Network

We are trying to build an interesting way to improve the ecosystem of the BTC network.

## Summary

Everything of Web3 starts from the decentralized ledger, and now we are considering whether we can build a new kind of decentralized ledger over different consensus spaces. In particular, we define a new global token protocol based on the UTXO transaction model and use the Bitcoin network and other currently stable blockchains as abstract nodes to record the states of the new global decentralized ledger together. As a result, the security of the new kind of token will be guaranteed by both the Bitcoin network and other blockchains like Ethereum, users can keep the integrity of their tokens and more diverse applications will be introduced into the BTC ecosystem as the application businesses can be deployed anywhere but the settlements are recorded on BTC.

Simply, the legitimacy of all on-chain states and operations can be equivalently verified and recorded simultaneously over different consensus spaces, regardless of where they were initiated. That’s why we call the new token protocol Omnize-UTXO.

## Motivation

We think the first thing to extend the BTC ecosystem is for assets issued on the BTC network to be able to circulate throughout the web3 world. But current methods have some problems.  
- The current paradigm(like token bridges) of using a token by wrapped it on multi-chains seperately lead to fragmentation, and may have some centralization and security issues related to the bridge.  
- If BTC was transferred to another chain through the current token bridge, and once the target chain breaks down, it will be very hard to correctly get the BTCs back to the users although locked in the bridge account.

The core of the Omnize-UTXO protocol is recording synchronized instead of bridging, even if all the other chains break down, as long as the Bitcoin network is still running, the user’s assets will not be lost.

- The fragment problem will be solved.
- The security of users' multi-chain assets can be greatly enhanced.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### O-UTXO

O-UTXO is short for the Omnize-UTXO. The UTXO transaction model is used for Omnize-UTXO, namely all tokens are stored in the unspent transaction outputs. To distinguish the UTXO of Omnize-UTXO from the UTXO of BTC, we call the former the `O-UTXO`. Anyone who has the private key to the O-UTXO can spend the O-UTXO.

#### Data structure of O-UTXO

The structure is like this
```js
{
    account: '<Public key of the owner of the O-UTXO>'
    amount: '<Token number in the O-UTXO>'
}
```

- The account is RECOMMENDED to be the compressed public key based on `secp256k1`

#### Index of O-UTXO

An O-UTXO is indexed by the O-TXID: O-INDEX, in which O-TXID is the id of the O-TX where the O-UTXO is generated, and the O-INDEX is the index of the O-UTXO in the transaction.

### O-TX

It is short for the transaction of Omnize-UTXO.

#### Data structure of O-TX

The structure is like this

```js
{
    deploy:  // Only used when deploying an Omnize-UTXO token
    {
        name: '<name of the token>',
        owner: '<account of the owner>'
    },
    name: "<name of the token>", // Can be absent when deploying
    inputs: [
        { // Can be absent when deploying
	        txid: '<O-TXID>',
            index: '<index of O-UTXO>',
            amount: '<amount of the O-UTXO>',
            signature: '<signature>'
        }
    ],
    outputs: [ // Can be absent when deploying
        <an O-UTXO>,
    ]  
}
```

The signature MAY be computed like this
sign(keccak256(CONCAT(BYTES(txid), BYTES(index), BYTES(amount))))

#### O-TX types

There are 3 types of O-TX
- Deploy: Deploy a new Omnize-UTXO token

    ```js
    {
        deploy: {
            name: '<name of the token>',
            owner: '<OWNER PK>'
        }
    }
    ```

- Mint: Mint new O-UTXOs

    ```js
    {
        name: '<name of the token> like: TEST-TOKEN',
        inputs: [
            {
                txid: 0x00,
   	            index: 0x00,
                amount: '<amount to be minted>'
                signature: '<signature>'
            }
        ],
        outputs: [
            <O-UTXO>,
        ]
    }
    ```

    - The `txid` is fixed to `0x00` in `Mint` operation
    - The `index` starts at 0, and increase by 1 after every minting

- Spend: Spend O-UTXOs

    ```js
    {
        name: '<name of the token>',
        inputs: [
            {
	            txid: '<O-TXID>',
                index: '<index of O-UTXO>',
                amount: '<amount of the O-UTXO>',
                signature: '<signature>'
            }
        ],
        outputs: [
            <O-UTXO>,
        ]
    }
    ```

#### O-TXID

It is short for Omnize-UTXO transaction ID and is generated according to the input O-UTXO indexes.  
O-TXID MAY be generated as following  

```js
keccak256(CONCAT(BYTES(inputs[0].txid, inputs[0].index, BYTES(inputs[0].amount),...))
```

#### Constraint

The balance amount of inputs MUST be equal to the balance amount of outputs

### Execution Layer

The main purpose of the Execution Layer is to guarantee all chains have executed the same transactions and have the same state. In addition, the Execution Layer can batch and execute Omnize-UTXO transactions(o-transactions for short) in a period, generate zk-proof, and commit it together with state changes to chains, to improve the performance of Omnize-UTXO.

### Interpreter

An interpreter SHOULD be introduced to execute O-TXs, due to the Bitcoin network not being able to deal with O-TXs. The transaction data that the interpreter executes are all recorded on the Bitcoin network, so anyone can restore the state of Omnize-UTXO tokens, to check if the interpreter functions well.

### Account Mapping Mechanism for Different Environments

In the simplest implementation, we can just build two mappings to get it. One is like `pk based on sece256k1 => account address in the special environment`, and the other is the reverse mapping.

The `Account System` on `Flow` is a typical example.  

- `Flow` has a built-in mechanism for `account address => pk`. The public key can be bound to an account (a special built-in data structure) and the public key can be obtained from the `account address` directly.  
- A mapping from `pk` to the `account address` on Flow can be built by creating a mapping `{String: Address}`, in which `String` denotes the data type to express the public key and the `Address` is the data type of the `account address` on Flow.  

## Rationale

### Architecture

#### Figure.1 Architecture

![image](https://github.com/Omniverse-Web3-Labs/some-writings/assets/83746881/ec4868c0-b657-425a-9bba-a0080b22199c)

With the Omnize-UTXO protocol, everyone can issue global tokens that can be used on multi-chains by leveraging scripts on Bitcoin, smart contracts, or similar mechanisms on other existing blockchains.

As shown in [Figure.1](#architecture).  

- The Omnize-UTXO smart contracts and the scripts of Bitcoin are referred to as **Abstract Nodes**. The states recorded by the Abstract Nodes that are stored on different blockchains respectively could be considered as copies of the global state, and they are ultimately consistent.  
- **Execution-Layer** is an off-chain execution program responsible for receiving Omnize-UTXO transactions, executing transactions, and generating proofs to chains.

### Principle

- The `UTXO transaction model` is mentioned [here](#UTXO transaction model). There are several advantages to the UTXO transaction model
    - No worry about transaction sequence, because UTXOs are executed serially
    - Privacy transactions can be easily supported
    - High concurrency, because one transaction can include many inputs and outputs
- The Execution Layer increases transaction capacity, saves gas fees, and reduces transaction latency.

#### Workflow

- Suppose there is
    - An Omnize-UTXO token O-TOKEN
    - A common user `A` has public key pk-A.
    - A common user `B` has public key pk-B.
    - An O-UTXO `A1` with O-TXID `txid`, O-INDEX 0
        ```js
        {
            account: 'pk-A',
            amount: 1000
        }
        ```

    - `A` signs `A1` and initiates an O-TX `tx-e`
        ```js
        {
            inputs: [
                {
                    txid: 'txid',
	                index: 0,
                    signature: 'the signature of A to A1'
                }
            ]
            outputs: [
                {
                    account: 'pk-A',
                    amount: 500
                },
                {
                    account: 'pk-B'
                    amount: 500
                }
            ]
        }  
        ```
    - `A` sends `tx-e` to the Execution Layer.
    - The Execution Layer checks that
        - If `A1` is valid, namely `A1` is not spent
        - If the inputs and outputs of `tx-e` match, the total amount of outputs is 1000
    - The Execution Layer executes `tx-e` after all checks are passed
    - Over a while, the Executor Layer batches transactions executed in the period, generates a related zk-proof and pushes transactions and proofs to L1s.  
    - L1s verify proofs
        - The interpreter gets proofs from the Bitcoin network, verifies proofs, and then updates states to show.
        - Smart contracts verify proofs and record new states.
    - Users can query outputs of `tx-e` on all chosen chains.

## More Information

More information can be found at [the GitHub of Omnize Labs](https://github.com/Omniverse-Web3-Labs).
