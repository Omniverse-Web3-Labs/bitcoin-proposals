# OMNI-UTXO for Bitcoin Network

We are trying to build an interesting way called `OMNI-UTXO protocol` to improve the ecosystem of the BTC network.

## Summary

Decentralized ledger techology(DLT) is the foundation of Web3, and now we are considering whether we can build a new kind of decentralized ledger over different consensus spaces of chains. In particular, we define a new global token protocol based on the UTXO transaction model and use the Bitcoin network and other currently stable blockchains as abstract nodes to record the states of the new global decentralized ledger together. As a result, the security of the new kind of token will be guaranteed by both the Bitcoin network and other blockchains like Ethereum, so that users can keep the integrity of their tokens and more diverse applications will be introduced into the BTC ecosystem as the application businesses can be deployed anywhere but the settlements are recorded on BTC.

Simply, the legitimacy of all on-chain states and operations can be equivalently verified and recorded simultaneously over different consensus spaces, regardless of where they were initiated. That’s why we call the new token protocol OMNI-UTXO.

## Motivation

We think the first thing to extend the BTC ecosystem is for assets issued on the BTC network to be able to circulate throughout the web3 world. But current methods have some problems.  
- The current paradigm(like token bridges) of using a token by wrapped it on multi-chains seperately lead to fragmentation, and may have some centralization and security issues related to the bridge.  
- If BTC was transferred to another chain through the current token bridge, and once the target chain breaks down, it will be very hard to correctly get the BTCs back to the users although locked in the bridge account.

The core of the OMNI-UTXO protocol is recording synchronized instead of bridging, even if all the other chains break down, as long as the Bitcoin network is still running, the user’s assets will not be lost.

- The fragment problem will be solved.
- The security of users' multi-chain assets can be greatly enhanced.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### O-TX

It is short for the transaction of OMNI-UTXO.

#### Data structure of O-TX

The structure is like this

```js
{
    deploy:  // Only used when deploying an OMNI-UTXO token
    {
        name: '<name of the token>',
        deployer: '<address of the deployer>'
    },
    name: "<name of the token>", // Can be absent when deploying
    signatures: ['<signature>'],
    inputs: [ // Can be absent when deploying
        {
            txid: '<txid of the transaction from which the input is generated>',
            index: '<index of the input in the transaction>',
            address: '<the omni-address of the owner>',
            amount: '<amount of the input>',
        }
    ],
    outputs: [ // Can be absent when deploying
        {
            address: '<owner address of the output>',
            amount: '<amount of the output>'
        },
    ]  
}
```

- The address is RECOMMENDED to be the compressed public key based on `secp256k1`
- The signature MAY be computed like this
    sign(HASH)
    - The data used to calculate HASH MUST include the transaction's inputs and outputs, it is RECOMMENDED to implement like this
    HASH = keccak256(CONCAT(BYTES(inputs), BYTES(outputs)))

#### O-TX types

There are 3 types of O-TX
- Deploy: Deploy a new OMNI-UTXO token

    ```js
    {
        deploy: {
            name: '<name of the token>',
            deployer: '<address of the deployer>'
        }
    }
    ```

    - After deployed, the `asset_id ` is RECOMMENDED to be created as following:

        ```js
        keccak256(CONCAT(BYTES(related native transaction hash), BYTES(output index), BYTES(name), BYTES(owner)))
        ```

- Mint: Mint new tokens

    ```js
    {
        name: '<name of the token> like: TEST-TOKEN',
        asset_id: '<the asset id created after deployed>',
        signatures: ['<signature>'],
        outputs: [
            {
                address: '<owner address of the output>',
                amount: '<amount of the output>'
            },
        ]
    }
    ```

    - The `txid` is fixed to `0x00` in `Mint` operation
    - The `index` starts at 0, and increase by 1 after every minting

- Spend: Spend O-UTXOs

    ```js
    {
        name: '<name of the token>',
        asset_id: '<the asset id created after deployed>',
        signatures: ['<signature>'],
        inputs: [
            {
                txid: '<txid of the transaction from which the input is generated>',
                index: '<index of the input in the transaction>',
                address: '<the omni-address of the owner>',
                amount: '<amount of the input>',
            }
        ],
        outputs: [
            {
                address: '<owner address of the output>',
                amount: '<amount of the output>'
            }
        ]
    }
    ```

#### O-TXID

It is short for OMNI-UTXO transaction ID and is generated according to the inputs of O-TX.  
O-TXID MAY be generated as following  

```js
keccak256(CONCAT(BYTES(inputs[0].txid, inputs[0].index, BYTES(inputs[0].amount),...))
```

### O-UTXO

O-UTXO is short for the UTXO of OMNI-UTXO.

The UTXO transaction model is used for OMNI-UTXO, namely all tokens are stored in the *unspent O-TX outputs*. To distinguish the UTXO of OMNI-UTXO from the UTXO of BTC, we call the former the `O-UTXO`. Anyone who has the private key to the O-UTXO can spend it.

#### Index of O-UTXO

An O-UTXO is indexed by the O-TXID: O-INDEX, in which O-TXID is the id of the O-TX from which the O-UTXO is generated, and the O-INDEX is the index of the O-UTXO in the transaction.

#### Constraint

The balance amount of inputs MUST be equal to the balance amount of outputs

### Execution Layer

The main purpose of the Execution Layer is to guarantee all chains have executed the same transactions and have the same state. In addition, the Execution Layer can batch and execute OMNI-UTXO transactions(o-transactions for short) in a period, generate zk-proof, and commit it together with state changes to chains, to improve the performance of OMNI-UTXO.

### Interpreter

An interpreter SHOULD be introduced to execute O-TXs, due to the Bitcoin network not being able to deal with O-TXs. The transaction data that the interpreter executes are all recorded on the Bitcoin network, so anyone can restore the state of OMNI-UTXO tokens, to check if the interpreter functions well.

## Rationale

### Architecture

#### Figure.1 Architecture

![image](https://github.com/Omniverse-Web3-Labs/some-writings/assets/83746881/ec4868c0-b657-425a-9bba-a0080b22199c)

With the Omni-UTXO protocol, everyone can issue global tokens that can be used on multi-chains by leveraging scripts on Bitcoin, smart contracts, or similar mechanisms on other existing blockchains.

As shown in [Figure.1](#architecture).  

- The OMNI-UTXO smart contracts and the scripts of Bitcoin are referred to as **Abstract Nodes**. The states recorded by the Abstract Nodes that are stored on different blockchains respectively could be considered as copies of the global state, and they are ultimately consistent.  
- **Execution-Layer** is an off-chain execution program responsible for receiving Omni-UTXO transactions, executing transactions, and generating proofs to chains.

### Principle

- The `UTXO transaction model` is mentioned [here](#UTXO transaction model). There are several advantages to the UTXO transaction model
    - No worry about transaction sequence, because UTXOs are executed serially
    - Privacy transactions can be easily supported
    - High concurrency, because one transaction can include many inputs and outputs
- The Execution Layer increases transaction capacity, saves gas fees, and reduces transaction latency.

#### Workflow

- Suppose there is
    - An OMNI-UTXO token O-TOKEN
    - A common user `A` has public key pk-A.
    - A common user `B` has public key pk-B.
    - An O-UTXO `A1` with O-TXID `txid`, O-INDEX 0
        ```js
        {
            address: 'pk-A',
            amount: 1000
        }
        ```

    - `A` signs `A1` and initiates an O-TX `tx-e`
        ```js
        {
            asset_id: '<the asset id created after deployed>',
            signatures: ['<signature of pk-A>'],
            inputs: [
                {
                    txid: 'txid',
                    index: 0,
                    address: 'pk-A',
                    amount: 1000,
                }
            ]
            outputs: [
                {
                    address: 'pk-A',
                    amount: 500
                },
                {
                    address: 'pk-B',
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

More information can be found at [the GitHub of Omni Labs](https://github.com/Omniverse-Web3-Labs).

## Contacts

- Email: chengjingxx@gmail.com
- Email: xiyuzheng1984@gmail.com
- Email: kay20475@hotmail.com
- Email: hht2015ily@gmail.com
