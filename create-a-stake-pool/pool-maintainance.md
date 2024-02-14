# Pool operations and maintenance

## Querying the stake snapshot

```bash
cardano-cli query stake-snapshot \
--testnet-magic 2 \
--stake-pool-id <pool_id>
```

## Querying the leadership schedule

```bash
cardano-cli query leadership-schedule \
--testnet-magic 2 \
--cold-verification-key-file cold.vkey \
--genesis shelley-genesis.json \
--vrf-signing-key-file vrf.skey \
--current
```

```bash
cardano-cli query leadership-schedule \
--testnet-magic 2 \
--cold-verification-key-file cold.vkey \
--genesis shelley-genesis.json \
--vrf-signing-key-file vrf.skey \
--next
```

## Checking the validity of your KES keys

```bash
cardano-cli query kes-period-info \
--testnet-magic 2 \
--op-cert-file opcert.cert
```

```
✓ Operational certificate's KES period is within the correct KES period interval
✓ The operational certificate counter agrees with the node protocol state counter
{
    "qKesCurrentKesPeriod": 107,
    "qKesEndKesInterval": 166,
    "qKesKesKeyExpiry": null,
    "qKesMaxKESEvolutions": 62,
    "qKesNodeStateOperationalCertificateNumber": 2,
    "qKesOnDiskOperationalCertificateNumber": 2,
    "qKesRemainingSlotsInKesPeriod": 7632249,
    "qKesSlotsPerKesPeriod": 129600,
    "qKesStartKesInterval": 104
}
```

## Renewing KES keys and the operational certificate

```bash
cardano-cli node key-gen-KES \
  --verification-key-file kes.vkey \
  --signing-key-file kes.skey
```

```bash
  cardano-cli node issue-op-cert --kes-verification-key-file kes.vkey \
  --cold-signing-key-file cold.skey \
  --operational-certificate-issue-counter-file opcert.counter \
  --kes-period <current kes period> \
  --out-file opcert.cert
```

Upload the new `kes.skey` and `opcert.cert` to your block-producing node.

## Withdrawing rewards

Build the stake address:

```bash
cardano-cli stake-address build \
--testnet-magic 2 \
--stake-verification-key-file stake.vkey \
--out-file stake.addr
```

Query its balance:

```bash
cardano-cli query stake-address-info \
--testnet-magic 2 \
--address $(cat stake.addr)
```

Example output:&#x20;

```
[
    {
        "address": "stake_test1urdflexy7e3uuflpvs0zycrc7zc8rtg9hguglvwq545j7mqf09vna",
        "delegation": "pool15r46gslrnhpe7ekvecfl895sm2pzhsxa8dhwh9ymgf06st3l86e",
        "rewardAccountBalance": 1834628277
    }
]
```

You can use `jq` to parse the output, for example, this `jq` command outputs 'address+balance', which will be useful later when building your transaction:

{% code overflow="wrap" %}
```bash
cardano-cli query stake-address-info \
--testnet-magic 2 \
--address $(cat stake.addr) | jq -r '.[0].address + "+" + (.[0].rewardAccountBalance|tostring)'
```
{% endcode %}

Output:

{% code overflow="wrap" %}
```
stake_test1upcezzdyhjcdcgdh28gy3w2xkfdxvs0hmee2p0v25l9uc8cgpaq2e+1579546084
```
{% endcode %}

\
Build a transaction to withdraw rewards, use `--witness-override 2.` It will be signed by stake and payment keys:

<pre class="language-bash"><code class="lang-bash">cardano-cli transaction build \
--testnet-magic 2 \
<strong>--witness-override 2 \
</strong><strong>--tx-in $(cardano-cli query utxo --address $(cat payment.addr) --testnet-magic 2 --out-file  /dev/stdout | jq -r 'keys[0]') \
</strong>--withdrawal $(cat stake.addr)+1834628277 \
--change-address $(cat payment.addr) \
--out-file withdraw-tx.raw
</code></pre>

Or, use this more automatic approach: 

```bash
cardano-cli transaction build \
--testnet-magic 2 \
--witness-override 2 \
--tx-in $(cardano-cli query utxo --address $(cat payment.addr) --testnet-magic 2 --out-file  /dev/stdout | jq -r 'keys[0]') \
--withdrawal $(cardano-cli query stake-address-info --testnet-magic 2 --address $(cat stake.addr) | jq -r '.[0].address + "+" + (.[0].rewardAccountBalance|tostring)') \
--change-address $(cat payment.addr) \
--out-file withdraw-tx.raw
```

```
cardano-cli transaction sign \
--tx-body-file withdraw_tx.raw  \
--signing-key-file payment.skey \
--signing-key-file stake.skey \
--testnet-magic 2 \
--out-file withdraw-tx.signed
```

```
cardano-cli transaction submit \
--tx-file withdraw-tx.signed \
--testnet-magic 2
```

## Changing pool parameters

You can update pool parameters using a new registration certificate. This time you will not pay the 500 ada deposit:&#x20;

```bash
cardano-cli stake-pool registration-certificate \
--testnet-magic 2 \
--cold-verification-key-file cold.vkey \
--vrf-verification-key-file vrf.vkey \
--pool-pledge <AMOUNT TO PLEDGE IN LOVELACE> \
--pool-cost <POOL COST PER EPOCH IN LOVELACE> \
--pool-margin <POOL OPERATOR MARGIN > \
--pool-reward-account-verification-key-file stake.vkey \
--pool-owner-stake-verification-key-file stake.vkey \
--pool-relay-ipv4 <RELAY NODE PUBLIC IP> \
--pool-relay-port <RELAY NODE PORT> \
--metadata-url <URL> \
--metadata-hash <POOL METADATA HASH> \
--out-file pool-registration.cert
```

Submit `pool-registration.cert` in a transaction.&#x20;

On a machine with a running node:

```
cardano-cli transaction build \
--testnet-magic 2 \
--tx-in $(cardano-cli query utxo --address $(cat payment.addr) --testnet-magic 2 --out-file  /dev/stdout | jq -r 'keys[0]') \
--change-address $(cat payment.addr) \
--witness-override 3 \
--certificate-file pool-registration.cert \
--out-file tx.raw
```

Sign the `tx.raw` on your air-gapped machine:

```
cardano-cli transaction sign \
--testnet-magic 2 \
--tx-body-file tx.raw \
--signing-key-file payment.skey \
--signing-key-file stake.skey \
--signing-key-file cold.skey \
--out-file tx.signed
```

Back on the machine with the running node, submit `tx.signed`:&#x20;

```
cardano-cli transaction submit \
--testnet-magic 2 \
--tx-file tx.sigend 
```

## Updating cardano-node and cardano-cli

1. Build cardano-node and cardano-cli on your local machine
2. Upload the new binaries to your node server:

```
scp cardano-node cardano-cli user@host:~/ 
```

3. Stop your node:

```
sudo systemctl stop cardano-node.service
```

4. Replace the old binaries with the new ones:

```
mv cardano* /usr/local/bin 
```

5. Restart the node:

```
sudo systemctl start cardano-node.service
```
