---
description: >-
  Cardano node v.8.0.0 introduced a new subset of commands to conduct polls
  among stake pool operators. A poll is official when it is signed by a genesis
  delegate key.
---

# Polls

{% hint style="success" %}
This tutorial requires [cardano-node v.8.0.0](https://github.com/input-output-hk/cardano-node/releases/tag/8.0.0).
{% endhint %}

{% hint style="info" %}

## Working with the polling feature on a local cluster

See this [tutorial](../module-8-setting-up-a-local-cluster/create-a-local-cluster-with-mkfiles-script.md) on how to run a local cluster using `mkfiles.sh` script provided in the cardano-node repository.

The examples below were created on a local cluster generated with the `mkfiles.sh` script. You should be able to replicate the examples on your own cluster.

For this, you need to produce the following after running your cluster:

1. **Build the payment address used to pay for the transaction fees:**

```bash
cardano-cli address build \
--payment-verification-key-file example/utxo-keys/utxo1.vkey \
--out-file example/utxo-keys/utxo1.addr \
--testnet-magic 42
```

2. **Get the hash of delegate 1 verification key:**

```
cardano-cli genesis key-hash \
--verification-key-file example/delegate-keys/delegate1.vkey > example/delegate-keys/delegate1.hash
```

3. **Get the pool ID of pool 1:**

```
cardano-cli stake-pool id \
--cold-verification-key-file example/pools/cold1.vkey \
--output-format hex > example/pools/pool.hex.id
```
{% endhint %}

## Creating a poll

To create a poll, run:

```
cardano-cli governance create-poll \
--question "Best name for a dog?" \
--answer "Cheeto" \
--answer "Ham" \
--answer "Rocco" \
--nonce 1 \
--out-file poll.cbor > poll.json
```

Note that `--nonce` is an optional but recommended option. It accepts a UINT, which serves as a unique identifier, allowing the same question to be asked at different times.

{% code overflow="wrap" %}
```
Poll created successfully.
Please submit a transaction using the resulting metadata.

{
    "94": {
        "map": [
            {
                "k": {
                    "int": 0
                },
                "v": {
                    "list": [
                        {
                            "string": "Best name for a dog?"
                        }
                    ]
                }
            },
            {
                "k": {
                    "int": 1
                },
                "v": {
                    "list": [
                        {
                            "list": [
                                {
                                    "string": "Cheeto"
                                }
                            ]
                        },
                        {
                            "list": [
                                {
                                    "string": "Ham"
                                }
                            ]
                        },
                        {
                            "list": [
                                {
                                    "string": "Rocco"
                                }
                            ]
                        }
                    ]
                }
            }
        ]
    }
}

Hint (1): Use '--json-metadata-detailed-schema' and '--metadata-json-file' from the build or build-raw commands.
Hint (2): You can redirect the standard output of this command to a JSON file to capture metadata.

Note that a serialized version of the poll suitable for sharing with participants has been generated at 'poll.cbor'.

```
{% endcode %}

Let's take a look at the serialized version of the poll:

```
cat poll.cbor

{
    "type": "GovernancePoll",
    "description": "An on-chain poll for SPOs: Best name for a dog?",
    "cborHex": "a1185ea200817442657374206e616d6520666f72206120646f673f0183816643686565746f816348616d8165526f63636f"
}
```

Participants (SPOs) will use the `poll.cbor` file to create and submit their responses.&#x20;

The _delegate-key-holder_ proposing the poll will publish the poll in a transaction. To build such a transaction, run: 

{% code overflow="wrap" %}
```
cardano-cli transaction build \
--babbage-era \
--testnet-magic 42 \
--tx-in $(cardano-cli query utxo --address $(cat utxo-keys/utxo1.addr) --testnet-magic 42 --out-file  /dev/stdout | jq -r 'keys[0]') \
--change-address $(cat utxo-keys/utxo1.addr) \
--metadata-json-file poll.json \
--json-metadata-detailed-schema \
--required-signer-hash $(cat delegate-keys/delegate1.hash) \
--out-file question.tx
```
{% endcode %}

{% hint style="info" %}
When building the transaction, you can use `--required-signer-hash` or&#x20;

`--required-signer`&#x20;

The example used `--required-signer-hash` because in a real-world scenario, the delegate signing keys are on cold storage and the build command requires access to a live node.

To get the hash of a delegate key, run:

```
cardano-cli genesis key-hash --verification-key-file delegate-keys/delegate1.vkey
0f455663bd57b2145bcea12302664a842bd4b8e69a1e05bb9f8e45ed
```
{% endhint %}

Sign the transaction with the delegate signing key and with a payment signing key to pay for the transaction fees:

```
cardano-cli transaction sign \
--tx-body-file question.tx \
--signing-key-file delegate-keys/delegate1.skey \
--signing-key-file utxo-keys/utxo1.skey \
--testnet-magic 42 \
--out-file question.tx.signed
```

{% hint style="info" %}
Note that in the example, the payment key is named **utxo1.skey.** This is the name given by the `mkfiles.sh` script. It is equivalent to a **payment.skey** that you might be familiar with.&#x20;
{% endhint %}

When inspecting the transaction (question.tx.signed):&#x20;

```
cardano-cli transaction view --tx-file question.tx.signed
```

you should see something like this: &#x20;

{% code overflow="wrap" %}
```
certificates: null
collateral inputs: []
era: Babbage
fee: 173729 Lovelace
inputs:
- 14bd4fe1f04829ed6f55350f21f243774e96f0ea9278ac8905e1a0e53dce0e01#0
metadata:
  '94':
  - - 0
    - - Best name for a dog?
  - - 1
    - - - Cheeto
      - - Ham
      - - Rocco
mint: null
outputs:
- address: addr_test1vzm3wju8c3u5w4vdq2ydes73vqz9kfx40j99cwjjvpefpmcg36av4
  address era: Shelley
  amount:
    lovelace: 299999826271
  datum: null
  network: Testnet
  payment credential key hash: b7174b87c47947558d0288dcc3d160045b24d57c8a5c3a52607290ef
  reference script: null
  stake reference: null
reference inputs: []
required signers (payment key hashes needed for scripts):
- 186c97a1194d2c62e3bfafef7b471cb0bb4ce09322ca44a908e81bee
return collateral: null
total collateral: null
update proposal: null
validity range:
  lower bound: null
  upper bound: null
withdrawals: null
witnesses:
- key: VKey (VerKeyEd25519DSIGN "240a1e444b9ed4d2080dd8ac731510037ea887dc1be26e8ab99d8c09b13864d9")
  signature: SignedDSIGN (SigEd25519DSIGN "8c92614f0fafd716fbdbb76ba83e6ac8e93d0c593acdc500328d8b0b7595d70f5a8cbbde499d9c0969ec41162f505a93e35d7dc3965f9a18d05a934b428de806")
- key: VKey (VerKeyEd25519DSIGN "da962fd3e59db09e724d698914a83bf7eacfbdccae07a78957c03b617f06e26e")
  signature: SignedDSIGN (SigEd25519DSIGN "83feb9bbd588d5933b25bad3da341043af088b5ab99e4f332e89f3e525c18a780909f1bc6da7d63372dda47ae901c63ecf23a62082efba62f35ecc424f5db20f")
```
{% endcode %}

Note that the **required signers** field includes the hash of the delegate key, and the **witnesses** field includes the delegate and payment key data.  &#x20;

Finally, submit the transaction as usual:

```
cardano-cli transaction submit \
--testnet-magic 42 \
--tx-file question.tx.signed
```

You can use _**db-sync**_ to check how it was registered online:

```
cexplorer=# SELECT * FROM tx_metadata WHERE key = 94;
```

```
 id | key |     json                                                              | bytes                                                                                                | tx_id 
----+-----+------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------+-------
 1  | 94  | {"0": ["Best name for a dog?"], "1": [["Cheeto"], ["Ham"], ["Rocco"]]} | \xa1185ea200817442657374206e616d6520666f72206120646f673f0183816643686565746f816348616d8165526f63636f | 11 
 (1 row)
```

```
cexplorer=# SELECT * FROM extra_key_witness;
```

```
 id |                            hash                            | tx_id
----+------------------------------------------------------------+-------
  1 | \x186c97a1194d2c62e3bfafef7b471cb0bb4ce09322ca44a908e81bee |    11
(1 row)
```

Of course, the hash matches the hash of the delegate key. This way, when SPOs see a poll signed with any of the delegate keys (verified by the delegate key hashes) they know this is an official poll.&#x20;

## Answering the poll

You can use the `answer-poll` command to create a response. Use the `--answer` option to immediately record your response by providing the index of your response. Alternatively, omit it to access the interactive method.&#x20;

Note that in any case, you are redirecting the output to `poll-answer.json`.

The proponents of the poll will have distributed the `poll.cbor` file from above. As an SPO, you will need it to answer the poll. It will look like this:

```
cat poll.cbor

{
    "type": "GovernancePoll",
    "description": "An on-chain poll for SPOs: Best name for a dog?",
    "cborHex": "a1185ea200817442657374206e616d6520666f72206120646f673f0183816643686565746f816348616d8165526f63636f"
}
```

### Using `--answer`

```
cardano-cli governance answer-poll \
--poll-file poll.cbor \
--answer 0 > poll-answer.json
```

This example votes for the option with index [0], Cheeto:

{% code overflow="wrap" %}
```
Best name for a dog?
→ Cheeto

Poll answer created successfully.
Please submit a transaction using the resulting metadata.
To be valid, the transaction must also be signed using a valid key
identifying your stake pool (e.g. your cold key).

Hint (1): Use '--json-metadata-detailed-schema' and '--metadata-json-file' from the build or build-raw commands.
Hint (2): You can redirect the standard output of this command to a JSON file to capture metadata.
```
{% endcode %}

## Responding interactively

```
cardano-cli governance answer-poll \
--poll-file poll.cbor > poll-answer.json
```

{% code overflow="wrap" %}
```
Best name for a dog?
[0] Cheeto
[1] Ham
[2] Rocco

Please indicate an answer (by index): 0

Poll answer created successfully.
Please submit a transaction using the resulting metadata.
To be valid, the transaction must also be signed using a valid key
identifying your stake pool (e.g. your cold key).


Hint (1): Use '--json-metadata-detailed-schema' and '--metadata-json-file' from the build or build-raw commands.
Hint (2): You can redirect the standard output of this command to a JSON file to capture metadata.
```
{% endcode %}

## Submitting the response in a transaction

```
cardano-cli transaction build \
    --babbage-era \
    --testnet-magic 42 \
    --tx-in $(cardano-cli query utxo --address $(cat utxo-keys/utxo1.addr) --testnet-magic 42 --out-file  /dev/stdout | jq -r 'keys[0]') \
    --change-address $(cat utxo-keys/utxo1.addr) \
    --metadata-json-file poll-answer.json \
    --json-metadata-detailed-schema \
    --required-signer-hash $(cat pools/pool.hex.id) \
    --out-file answer.tx
```

```
cardano-cli transaction sign \
--tx-body-file answer.tx \
--signing-key-file pools/cold1.skey \
--signing-key-file utxo-keys/utxo1.skey \
--testnet-magic 42 \
--out-file answer.tx.signed
```

When inspecting the signed transaction, **required signers** should contain the cold key hash (the pool ID), and **witnesses** should contain both the cold and payment key data:&#x20;

```
cardano-cli transaction view --tx-file answer.tx.signed
```

{% code overflow="wrap" %}
```
auxiliary scripts: null
certificates: null
collateral inputs: []
era: Babbage
fee: 173377 Lovelace
inputs:
- a218b5e10b821a407b82378daaef83991672ecf0b8eca234569cba73efde1000#0
metadata:
  '94':
  - - 2
    - '"@%\184\148''\143s\204\DEL\129\156=}\230\231\149W>,\ENQ\SYN7q\170S\218\196\189ij\198`"'
  - - 3
    - 0
mint: null
outputs:
- address: addr_test1vzm3wju8c3u5w4vdq2ydes73vqz9kfx40j99cwjjvpefpmcg36av4
  address era: Shelley
  amount:
    lovelace: 299999652894
  datum: null
  network: Testnet
  payment credential key hash: b7174b87c47947558d0288dcc3d160045b24d57c8a5c3a52607290ef
  reference script: null
  stake reference: null
reference inputs: []
required signers (payment key hashes needed for scripts):
- 939b194da1f5620da299f855ca9dca0c5a6b33d2584b7164cc69a3f7
return collateral: null
total collateral: null
update proposal: null
validity range:
  lower bound: null
  upper bound: null
withdrawals: null
witnesses:
- key: VKey (VerKeyEd25519DSIGN "e292c1986cc4882dc187258526da83b80b9905104d661bb9ee3aca81aafe9c9d")
  signature: SignedDSIGN (SigEd25519DSIGN "7a5ca4aab04e58d7d32e5e9ce8cea5db260997e69d2f287d67e2d4421cb6ef9353ebaf18558f6a08604daa18c2b4c81006fa7d30a3f1ffdb8accb0f09a99580f")
- key: VKey (VerKeyEd25519DSIGN "da962fd3e59db09e724d698914a83bf7eacfbdccae07a78957c03b617f06e26e")
  signature: SignedDSIGN (SigEd25519DSIGN "f99334fa305ea2941b631ab7b2c5badd57234045b6d181dbbb11b68a3c584e80d2bcb56f76a01436e2b5635c43bbeb25fabc23578c2dccac21070d004ceb5602")
```
{% endcode %}

```
cardano-cli transaction submit \
--testnet-magic 42 \
--tx-file answer.tx.signed
```

You can use _**db-sync**_ again to track responses:

```
SELECT * FROM tx_metadata WHERE key = 94;
```

```
 id | key |                                        json                                         |                                                bytes                                                 | tx_id
----+-----+-------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------+-------
  1 |  94 | {"0": ["Best name for a dog?"], "1": [["Cheeto"], ["Ham"], ["Rocco"]]}              | \xa1185ea200817442657374206e616d6520666f72206120646f673f0183816643686565746f816348616d8165526f63636f |    11
  2 |  94 | {"2": "0x4025b894278f73cc7f819c3d7de6e795573e2c05163771aa53dac4bd696ac660", "3": 0} | \xa1185ea20258204025b894278f73cc7f819c3d7de6e795573e2c05163771aa53dac4bd696ac6600300                 |    12
(2 rows)
```

```
SELECT * FROM extra_key_witness;
```

```
 id |                            hash                            | tx_id
----+------------------------------------------------------------+-------
  1 | \x186c97a1194d2c62e3bfafef7b471cb0bb4ce09322ca44a908e81bee |    11
  2 | \x939b194da1f5620da299f855ca9dca0c5a6b33d2584b7164cc69a3f7 |    12
(2 rows)
```

## Verifying answers <a href="#verifying-answers" id="verifying-answers"></a>

Finally, it’s possible to verify answers seen on-chain using the `governance verify-poll` command. ‘Verify’ means:

* It checks that an answer is valid within the context of a given survey
* It returns the list of signatories key hashes found in the transaction;\
  in the case of a valid submission, one key hash will correspond to a known\
  stake pool ID.

```
cardano-cli governance verify-poll \
--poll-file poll.cbor \
--tx-file answer.tx.signed
Found valid poll answer with 1 signatories
[
    "939b194da1f5620da299f855ca9dca0c5a6b33d2584b7164cc69a3f7"
]
```

## Using db-sync on your test

Note that the `configuration.yaml` file, produced by the `mkfiles.sh` script, does not contain the hashes of the genesis files, but db-sync demands them. Therefore, if you want to use _db-sync_, you will need to manually add the hashes of Byron, Shelley and Alonzo genesis files to the `example/configuration.yaml`file:

```
cardano-cli byron genesis print-genesis-hash \
--genesis-json example/genesis/byron/genesis.json
f80aef63f3ad1dbd52ac08d783e27f3d94d10ce6904001a0ff98a9f3fb2a1592
```

```
cardano-cli genesis hash \
--genesis example/genesis/shelley/genesis.json
e139930b28e1dbf196f717e3ea8b8e79606a5e64c2d513c1e455a4b20cba3419
```

```
cardano-cli genesis hash \
--genesis example/genesis/shelley/genesis.alonzo.json
b9c80a36b643e1b0be59e5aabe9d5ca705635ac4583ff217ce7283ad96fd7d5c
```

Then, add the hashes to the configuration file (by default it only contains the paths):

```
...
ByronGenesisFile: genesis/byron/genesis.json
ByronGenesisHash: f80aef63f3ad1dbd52ac08d783e27f3d94d10ce6904001a0ff98a9f3fb2a1592
ShelleyGenesisFile: genesis/shelley/genesis.json
ShelleyGenesisHash: e139930b28e1dbf196f717e3ea8b8e79606a5e64c2d513c1e455a4b20cba3419
AlonzoGenesisFile: genesis/shelley/genesis.alonzo.json
AlonzoGenesisHash: b9c80a36b643e1b0be59e5aabe9d5ca705635ac4583ff217ce7283ad96fd7d5c
ConwayGenesisFile: genesis/shelley/genesis.conway.json
...
```

Finally, when starting db-sync, you might see complaints about StartTime mismatch. If this happens, please make sure to replace the `StartTime` on `Shelley Genesis` with the value from `StartTime` on `Byron Genesis.` You should be good to go.
