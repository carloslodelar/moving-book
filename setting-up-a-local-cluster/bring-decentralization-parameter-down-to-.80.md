---
cover: ../.gitbook/assets/cluster (1).png
coverY: 0
---

# Bringing the decentralization parameter down to .80

For your pool to produce blocks, you need to lower the decentralization parameter. To do that, create an update proposal to lower the decentralization parameter.  

First, generate non-extended verification keys for your genesis delegates:  

```bash
cardano-cli key non-extended-key \
--extended-verification-key-file genesis-keys/shelley.000.vkey \
--verification-key-file genesis-keys/non.e.shelley.000.vkey
cardano-cli key non-extended-key \
--extended-verification-key-file genesis-keys/shelley.001.vkey \
--verification-key-file genesis-keys/non.e.shelley.001.vkey
```

Update proposals need to be submitted during the first 4k/f slots of the epoch. Keep in mind that Shelley epochs have 20 times as many slots as Byron epochs. This short script will show if there is time to submit the update proposal in the current epoch.  Change the value of Byron slots to be subtracted from the current tip (in this case it was 1350):  

```bash
#!/usr/bin/env bash

BYRON_SLOTS=1350
TIP=$(cardano-cli query tip --testnet-magic 42 | jq .slot)
SHELLEY_SLOTS=$(($TIP-$BYRON_SLOTS))
SHELLEY_EPOCH_LENGTH=$(cat configuration/shelley-genesis.json | jq .epochLength)
K=$(cat configuration/shelley-genesis.json | jq .securityParam)
F=$(cat configuration/shelley-genesis.json | jq .activeSlotsCoeff)

UPDATE_PROPOSAL_TH=$(echo "4*$K/$F" | bc)
SLOT_IN_EPOCH=$(($SHELLEY_SLOTS % $SHELLEY_EPOCH_LENGTH))

echo "UPDATE-THRESOLD: $UPDATE_PROPOSAL_TH"
echo "SLOT IN EPOCH: $SLOT_IN_EPOCH"
```

```bash
chmod +x whereinepoch.sh
```

You need to submit the proposal in the first 4k/f slots in the epoch, so before slot 3600 of the current epoch:  

<pre class="language-bash"><code class="lang-bash"><strong>./whereinepoch.sh
</strong>UPDATE-THRESOLD: 3600
SLOT IN EPOCH: 320
</code></pre>

You can now create the proposal like this:

```bash
cardano-cli governance create-update-proposal \
--out-file transactions/update.D80.proposal \
--epoch $(cardano-cli query tip --testnet-magic 42 | jq .epoch) \
--genesis-verification-key-file genesis-keys/non.e.shelley.000.vkey \
--genesis-verification-key-file genesis-keys/non.e.shelley.001.vkey \
--decentralization-parameter 80/100
```

```bash
CHANGE=$(($(cardano-cli query utxo --address $(cat pool1/payment.addr) --testnet-magic 42 --out-file  /dev/stdout | jq -cs '.[0] | to_entries | .[] | .value.value') - 1000000))
```

```bash
cardano-cli transaction build-raw \
--shelley-era \
--fee 1000000 \
--invalid-hereafter $(expr $(cardano-cli query tip --testnet-magic 42 | jq .slot) + 1000) \
--tx-in $(cardano-cli query utxo --address $(cat pool1/payment.addr) --testnet-magic 42 --out-file  /dev/stdout | jq -r 'keys[]') \
--tx-out $(cat pool1/payment.addr)+$CHANGE \
--update-proposal-file transactions/update.D80.proposal \
--out-file transactions/update.D80.proposal.txbody
```

```bash
cardano-cli transaction sign \
--tx-body-file transactions/update.D80.proposal.txbody \
--signing-key-file pool1/payment.skey \
--signing-key-file bft0/shelley.000.skey \
--signing-key-file bft1/shelley.001.skey \
--out-file transactions/update.D80.proposal.txsigned
```

```bash
cardano-cli transaction submit --testnet-magic 42 \
--tx-file transactions/update.D80.proposal.txsigned
```

So, with the proposal submitted on epoch 5, the node logs show that the update proposal will take effect at epoch 6:

{% code overflow="wrap" %}
```
Event: LedgerUpdate (HardForkUpdateInEra S (Z (WrapLedgerUpdate {unwrapLedgerUpdate = ShelleyUpdatedProtocolUpdates [ProtocolUpdate {protocolUpdateProposal = UpdateProposal {proposalParams = PParams {_minfeeA = SNothing, _minfeeB = SNothing, _maxBBSize = SNothing, _maxTxSize = SNothing, _maxBHSize = SNothing, _keyDeposit = SNothing, _poolDeposit = SNothing, _eMax = SNothing, _nOpt = SNothing, _a0 = SNothing, _rho = SNothing, _tau = SNothing, _d = SJust (4 % 5), _extraEntropy = SNothing, _protocolVersion = SNothing, _minUTxOValue = SNothing, _minPoolCost = SNothing}, proposalVersion = Nothing, proposalEpoch = EpochNo 6}, protocolUpdateState = UpdateState {proposalVotes = [KeyHash "5feef96101e0386c56f7661e46438642156f8d1af7e8fec92b1a402a",KeyHash "b8b358d5093f6d1e3a5128a36fe70031c15b7d57817e44f7460142a0"], proposalReachedQuorum = True}}]})))
```
{% endcode %}

The stake pool starts producing blocks on epoch 6:

{% code overflow="wrap" %}
```
[CLR:cardano.node.Forge:Info:32] [2022-12-01 05:00:52.80 UTC] fromList [("credentials",String "Cardano"),("val",Object (fromList [("block",String "1a0702eb290715e76193097e796e8379e54b51337377e0fef27e3015f29802ef"),("blockNo",Number 12286.0),("blockPrev",String "20b92ebfc6218e985bd84790343e9406e92321f78e12350eeb760cde6351a53e"),("kind",String "TraceForgedBlock"),("slot",Number 19470.0)]))]

```
{% endcode %}
