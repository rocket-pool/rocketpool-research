# Specification for Rewards Calculation

This document serves as a formal specification for the way that the rewards intervals and the values within are calculated as part of the [Redstone](https://medium.com/rocket-pool/rocket-pool-the-merge-redstone-601d9efd6b4) rewards system.

## Disclaimers / Notes

- All arithmetic described here is intended to be **integer arithmetic**. There are no floating point values, and floating point division is not used to allow for maximum portability and elimination of floating point errors.
- Unless explicitly specified, the following rules about data formats apply:
  - Timestamps are represented as **Unix timestamps**, and are provided as a **total number of seconds** since the Unix Epoch.
  - Token quantities are represented in **wei**.
  - The code samples here are all presented in **pseudocode** format and will not compile to a known language. Extrapolate to your system of choice accordingly.


## Scheduling and Target Blocks

As with the original protocol, rewards in Redstone are minted and recorded in discrete **intervals**.
Each interval is determined by an on-chain setting that specifies the amount of time that must pass from the previous interval until the next interval is ready to be generated.

### Time of Eligibility

This value marks the **start time**, called `startTime` (as a **Unix timestamp**, the number of seconds since the Unix Epoch) of the currently active interval:
```go
RocketRewardsPool.getClaimIntervalTimeStart()
```

This value marks the **amount of time** in seconds, called `intervalTime`, that must pass since `startTime` before a new interval is ready to be recorded:
```go
RocketRewardsPool.getClaimIntervalTime()
```

The current time is determined by the `timestamp` field in the **header of the latest block** of the Execution client - also a Unix timestamp.
When the current time is greater than `intervalTime` plus `startTime`, a new interval is eligible.

More specifically, the number of eligible intervals at any given time is calculated by the latest block's `timestamp`, minus `startTime`, divided by `intervalTime` (using integer division):

```go
latestBlockTime := latestBlockHeader.Time
timeSinceStart := latestBlockTime - startTime
intervalsPassed := timeSinceStart / intervalTime
```

This quantity is multiplied by `intervalTime` and added to `startTime` to produce the most recent eligibility timestamp, known as `endTime`:

```go
endTime := startTime + (intervalTime * intervalsPassed)
```

`endTime` is used to determine which **Beacon slot** and **Execution block** to snapshot the states for when calculating rewards (see below).


### Missed / Multi-Period Intervals

If the number of `intervalsPassed` is greater than 1, it indicates that the Oracle DAO failed to adequately report a rewards checkpoint during one interval window and has entered into the window for a subsequent interval.

When this happens, RPL is still minted on schedule (e.g., at the start of the new interval window).
The missed rewards checkpoint is simply rolled into the new one.
RPL and ETH rewards pending distribution have accumulated during this time, so the rewards checkpoint will use these pending figures as the amounts to distribute as part of the interval and add them to the Merkle Tree accordingly.

In other words, "missing" a rewards interval does not cause a loss of rewards.
It merely delays their distribution.

The number of intervals passed is recorded in the event emitted during a rewards tree submission to account for this occurrence.


### Target Beacon Slot

Each rewards period is ultimately defined by a target **Beacon slot** and corresponding **Execution block** which serve as chain state reference points for rewards calculation.

Once the system detects that a new rewards interval is eligible, it will calculate the **Beacon Epoch** that `endTime` falls in using the Beacon chain's genesis configuration.
The slot used is the first slot *after* `endTime`; that is, the first slot where `slotTime > endTime`.

```go
genesisTime := eth2Config.GenesisTime // Unix time of Beacon chain's genesis
secondsPerSlot := eth2Config.SecondsPerSlot
slotsPerEpoch := eth2Config.SlotsPerEpoch

totalTimespan := endTime - genesisTime
targetBcSlot := math.Ceil(totalTimespan / secondsPerSlot) // First slot *after* endTime
targetSlotEpoch := targetBcSlot / slotsPerEpoch
```

Once the target Epoch is determined, the target `slot` becomes that **last slot in that epoch**.
If that slot was missed (it has an empty Block because the proposer failed), then the target slot will become the **previous slot**.
If that is also missed, use the slot before that, and so on until the slot was not missed.

This slot will become the `targetBcSlot` which will be used for state snapshotting.

The intent is to use the **latest state of the Beacon chain prior to the Epoch boundary**, as the Epoch boundary signals the beginning of the next interval. 


### Target Execution Block

Once the `targetBcSlot` has been found, the corresponding block on the Execution layer can be determined.

Pre-merge, this will be the **last block before `targetSlotEpoch`'s timestamp**.
Note that this could potentially be an EL block that was added to the Execution chain *after* `targetBcSlot` was added to the Beacon chain.

Post-merge, this will be the Execution block corresponding to `targetBcSlot`.

This block will be the `targetElBlock`.


## Timing of State Collection

Once `targetBcSlot` and `targetElBlock` have been determined, the user will need to **wait until the Epoch after `targetSlotEpoch` has been finalized**.
This is because the rewards calculation will involve analyzing the attestation performance of validators up to the `targetBcSlot`.
As an attestation can be sent up to 32 slots after the assigned slot, the next Epoch must also be finalized so the attestation performance can be tracked.

For example, if a rewards interval occurred on Epoch 63, the user would have to wait until Epoch 64 was finalized before generating the rewards tree.

*Note that this relies on the ability to query the state of both `targetBcSlot` and `targetElBlock` well past their submission timestamps; users may need access to archive nodes if their clients cannot look far enough back.*


## RPL Rewards

This section describes the calculation for the RPL rewards distributed to each node operator and the Protocol DAO treasury.


### RPL Amounts per Group

The amount of RPL to be distributed during a rewards checkpoint can be found with the following contract function:

```go
RocketRewardsPool.getPendingRPLRewards()
```

This accounts for all of the RPL minted since the last rewards interval submission.
If multiple intervals have gone by and are being rolled up into this period, all of the RPL minted across each of them will be inherently aggregated into this value.

RPL rewards are divided into three groups, the fraction of which can be retrieved by the following contract methods:
- Collateral rewards for normal Node Operators
    ```go
    collateralPercent := RocketRewardsPool.getClaimingContractPerc("rocketClaimNode")
    ```

- Oracle DAO rewards
    ```go
    oDaoPercent :=RocketRewardsPool.getClaimingContractPerc("rocketClaimTrustedNode")
    ```

- Protocol DAO (treasury) rewards
    ```go
    pDaoPercent := RocketRewardsPool.getClaimingContractPerc("rocketClaimDAO")
    ```

Each of these values is a percentage, given in **wei**, where 100% (1.0) corresponds to 1 x 10^18 wei.
For example, if the collateral fraction `collateralPercent` had a value of `700000000000000000` (7 x 10^17), this corresponds to a percentage value of 70% (0.7).

Thus, the total expected amount per group is as follows:

```go
_100Percent := 1e18
pendingRewards := RocketRewardsPool.getPendingRPLRewards()

collateralRewards := pendingRewards * collateralPercent / _100Percent
oDaoRewards := pendingRewards * oDaoPercent / _100Percent
pDaoRewards := pendingRewards * pDaoPercent / _100Percent
```
    
Note that these will not be the **final** values attributed to each of these groups due to division truncation; they are simply starting points when calculating the actual values per group.
The final amounts are discussed later in this section. 


### Collateral Rewards

Start by acquiring the complete list of node addresses using the following contract functions:

```go
nodeCount := RocketNodeManager.getNodeCount()
nodeAddresses := address[nodeCount]
for i = 0; i < nodeCount; i++ {
    nodeAddresses[i] = RocketNodeManager.getNodeAt(i)
}
```

For each node, retrieve the **effective RPL stake**:

```go
nodeEffectiveStake := RocketNodeStaking.getNodeEffectiveRPLStake(nodeAddress)
```

This returns the effective amount of RPL, in **wei**, that is staked by the node.
The logic surrounding this value (such as the state of each minipool in the node) is handled by the smart contracts, so this value does not need to be modified.

Next, scale the `nodeEffectiveStake` by how long the node has been registered.
This prorates RPL rewards for new nodes that haven't been active for a full rewards interval, so they only receive a corresponding fraction of the rewards based on how long they've been registered.

For example, if the rewards period were 6300 Epochs (28 days) and a node registered 10 days ago, their `nodeEffectiveStake` would be reduced to **35.7%** (10 / 28) of its true value.

The node's registration time can be retrieved with the following contract method:

```go
registrationTime := RocketNodeManager.getNodeRegistrationTime(nodeAddress)
```

This should be subtracted from the timestamp of `targetElBlock` to determine the node's age.
It should then be compared to `intervalTime` to determine the prorated effective stake:

```go
nodeAge := targetElBlock.Timestamp - registrationTime
if (nodeAge < intervalTime) {
    nodeEffectiveStake = nodeEffectiveStake * nodeAge / intervalTime
}
```

When finished, add each of these to retrieve the `totalEffectiveRplStake` across the entire network.

With this in hand, you can now calculate the **collateral RPL per node** by taking the original `collateralRewards` value, multiplying by the `nodeEffectiveStake`, and dividing by the `totalEffectiveRplStake`:

```go
nodeCollateralAmount := collateralRewards * nodeEffectiveStake / totalEffectiveRplStake
```

Sum the `nodeCollateralAmount` for each node to arrive at the `totalCalculatedCollateralRewards`.
As a sanity check, compare this to the original `collateralRewards` value using the **total number of minipools** as an delta value:

```go
if collateralRewards - totalCalculatedCollateralRewards > numberOfMinipools {
    // Raise an error because the calculation has excessive error
}
```


### Oracle DAO Rewards

Start by acquiring the list of Oracle DAO node addresses using the following contract methods:

```go

oDaoCount := RocketDAONodeTrusted.getMemberCount()
oDaoAddresses := address[oDaoCount]
for i = 0; i < oDaoCount; i++ {
    oDaoAddresses[i] = RocketDAONodeTrusted.getMemberAt(i)
}
```

Next, scale the amount of RPL earned by each Oracle DAO node by how long the node has been registered.
This prorates RPL rewards for new nodes that haven't been active for a full rewards interval, so they only receive a corresponding fraction of the rewards based on how long they've been registered.

For example, if the rewards period were 6300 Epochs (28 days) and a node registered 10 days ago, their share would be reduced to **35.7%** (10 / 28) of its true value.

The node's registration time can be retrieved with the following contract method:

```go
registrationTime := RocketNodeManager.getNodeRegistrationTime(nodeAddress)
```

This should be subtracted from the timestamp of `targetElBlock` to determine the node's age.
It should then be compared to `intervalTime` to determine the prorated rewards.

One way to accomplish this is to use the **number of seconds** the node participated in the interval as an analog to the "effective stake" in the Collateral RPL calculation above:

```go
nodeAge := targetElBlock.Timestamp - registrationTime
participatedSeconds := intervalTime
if (nodeAge < intervalTime) {
    participatedSeconds = nodeAge
}
```

When finished, add each of these to retrieve the `totalParticipatedSeconds` for all oDAO nodes.

With this in hand, you can now calculate the **Oracle DAO RPL per node** by taking the original `oDaoRewards` value, multiplying by the `participatedSeconds`, and dividing by the `totalParticipatedSeconds`:

```go
oDaoAmount := oDaoRewards * participatedSeconds / totalParticipatedSeconds
```

Sum the `oDaoAmount` for each node to arrive at the `totalCalculatedODaoRewards`.
As a sanity check, compare this to the original `oDaoRewards` value using the **total number of minipools** as an delta value:

```go
if oDaoRewards - totalCalculatedODaoRewards > numberOfMinipools {
    // Raise an error because the calculation has excessive error
}
```


### Protocol DAO Treasury Rewards

Unlike the other two groups, the amount of RPL awarded to the Protocol DAO Treasury is not calculated on its own.
Rather, the Protocol DAO Treasury is simply used as a buffer for all of the remaining RPL that's unaccounted for:

```go
actualPDaoRewards := pendingRewards - totalCalculatedCollateralRewards - totalCalculatedODaoRewards
```

You may want to compare this amount to the original `pDaoRewards` calculation earlier to log how much of a delta is being handled, but the sanity checks in the previous two steps will prevent this from being too far from the expected value.


## Smoothing Pool Rewards

The Smoothing Pool's current balance is distributed to all of the nodes that have been **opted into the Smoothing Pool** for some (or all) of this rewards interval and are **eligible**, with one exception.


### Interval 0

The **first rewards interval** using the Redstone rewards system **will not produce Smoothing Pool rewards**.
This is because the Smoothing Pool's rewards calculation depends on the time that the previous interval occurred as a way to determine each minipool's eligibility and prorating status, and the event containing that data is only emitted upon a successful rewards snapshot.

For the first interval using Redstone's system (interval 0), ignore the Smoothing Pool calculation.
Its balance will be rolled over into interval 1.


### Balance and Start Blocks

Start by getting the current balance of the Smoothing Pool contract:

```go
smoothingPoolBalance := RocketSmoothingPool.Balance()
```

If the balance is 0 (e.g., because nobody has opted into the Smoothing Pool), simply end here.

Next, get the rewards event emitted for the previous interval:

```go
previousIntervalEvent := RocketRewardsPool.RewardSnapshot(currentIndex - 1)
```

From this event, you can get the `bnStartBlock` and the `elStartBlock` for this interval.

The `bnStartBlock` is the first non-missed slot in the Epoch *after* the Epoch that `previousIntervalEvent.ConsensusBlock` belonged to.

Pre-merge, the `elStartBlock` is simply `previousIntervalEvent.ExecutionBlock + 1`.
Post-merge, the `elStartBlock` is the EL block that corresponds to `bnStartBlock`.

This makes the Smoothing Pool's duration for this interval equal to:

```go
totalIntervalDuration := targetElBlock.Time / elStartBlock.Time
```


### Node Eligibility

For each registered node (the gathering of which was shown previously in the RPL calculation), assess its eligibility for Smoothing Pool rewards.

Get the opt-in status and the last time of a Smoothing Pool status change for the node:
```go
isOptedIn := RocketNodeManager.getSmoothingPoolRegistrationState(nodeAddress)
statusChangeTime := RocketNodeManager.getSmoothingPoolRegistrationChanged(nodeAddress) // Timestamp of the last opt-in or opt-out operation
```

If the node wasn't opted in **at all** (at any point) during this interval, it is not eligible.
Ignore it.

If the node was opted in **for the entire period**, it is fully eligible for rewards:

```go
nodeEligibleSeconds = totalIntervalDuration
```

If the node opted in **during the period**, it is *partially* eligible.
Calculate its eligibility factor as follows:

```go
nodeEligibleSeconds = (targetElBlock.Time - statusChangeTime) / totalIntervalDuration
```

If the node opted out **during the period**, it is *partially* eligible.
Calculate its eligibility factor as follows:

```go
nodeEligibleSeconds = (statusChangeTime - elStartBlock.Time) / totalIntervalDuration
```

Next, look at the minipools for the node with the following contract methods:
```go
nodeMinipoolCount := RocketMinipoolManager.getNodeMinipoolCount(nodeAddress)
nodeMinipools := address[nodeMinipoolCount]
for i := 0; i < nodeMinipoolCount; i++ {
    nodeMinipools[i] = RocketMinipoolManager.getMinipoolAt(i)
}
```

Get the current `state` and `penaltyCount` for each minipool:

```go
state := minipool.getStatus()
penaltyCount := RocketStorage.getUint(keccak256("network.penalties.penalty", minipoolAddress))
```

If the `state` is `staking` and the `penaltyCount` is **3 or more**, this node is a cheater and is not eligible.
Remove it from the list of eligible nodes and ignore it.


### Calculating Attestation Performance

For each `staking` minipool in each eligible node, process the attestation performance of the minipool to gauge its `participationRate`.

Attestation performance is calculated on an Epoch-by-Epoch basis, from the first Epoch to the last Epoch of the interval, as follows for each Epoch:

1. Get the **attestation committees** for the Epoch (e.g., `/eth/v1/beacon/states/head/committees?epoch=<epochIndex>`)
2. Traverse the list of slots and committees, noting the `slotIndex`, `committeeIndex`, and `position` of an attestation assignment for the minipool.
3. Get the block at `slotIndex` (e.g., `/eth/v2/beacon/blocks/<slotIndex>`)
4. Look at the attestations for the matching `slotIndex`, `committeeIndex`, and `position`.
   1. If it was recorded, this attestation was successful. Add it to a running list of `goodAttestations`.
   2. If not, try the next slot. Continue doing this for up to 1 Epoch away (`BeaconConfig.SlotsPerEpoch`). If it isn't in any of them, the attestation was missed. Add it to a running list of `missedAttestations`.


### Calculating Node Rewards

Start by calculating the average fee (commission) across all of the eligible minipools, which can be retrieved with the following contract function per minipool:

```go
minipoolFee := minipool.getNodeFee()
```

Call it the `averageFee`.

Next, calculate the amount of ETH that should go to the pool stakers (`poolStakerShare`), and the amount that goes to all of the node operators (`nodeOpShare`) based on this average fee:

```go
halfSmoothingPoolBalance := smoothingPoolBalance / 2
commissionAmount := halfSmoothingPoolBalance * averageFee / _100Percent
poolStakerShare := halfSmoothingPool - commissionAmount
nodeOpShare := smoothingPoolBalance - poolStakerShare
```

As with RPL rewards, these are not the final values - they simply serve as a reference when calculating each node operator's share.

For each eligible minipool, start with a base `minipoolShare` of 100% plus its fee (for example, a 15% commission minipool would start with a base share of 115):

```go
minipoolShare := _100Percent + minipoolFee
```

Now, scale this by its `eligibilityFactor` (the amount of time it was opted into the Smoothing Pool):

```go
if nodeEligibleSeconds < totalIntervalDuration {
    minipoolShare = minipoolShare * nodeEligibleSeconds / totalIntervalDuration
}
```

Next, scale this by its `participationRate` (the ratio of `goodAttestations` to total attestations):

```go
minipoolShare = minipoolShare * goodAttestations / (goodAttestations + missedAttestations)
```

Add up all of the individual `minipoolShare` values to determine the `totalMinipoolShare`.

Using this, the share of the Smoothing Pool ETH per minipool can be calculated as follows:

```go
minipoolEth := nodeOpShare * minipoolShare / totalMinipoolShare
```

Add up the `minipoolEth` for each node's minipools.
This is the **total amount of ETH awarded to that node:** `nodeSmoothingPoolEth`.

Add up the `minipoolEth` for each minipool.
This is the **total amount of ETH awarded to all Node Operators:** `totalEthForMinipools`.

Calculate the true value of the ETH awarded to pool stakers:

```go
truePoolStakerEth := smoothingPoolBalance - totalEthForMinipools
```


## Constructing the Tree

With all of the above values, you can now create the Merkle tree for this interval using the [tree specification](./merkle-tree-spec.md).