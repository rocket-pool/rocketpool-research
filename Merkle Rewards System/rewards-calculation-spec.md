# Specification for Rewards Calculation

This document serves as a formal specification for the way that the rewards intervals and the values within are calculated as part of the [Redstone](https://medium.com/rocket-pool/rocket-pool-the-merge-redstone-601d9efd6b4) rewards system.


## Version

This describes **v8** of the rewards calculation ruleset.

This version implements RPIP-30 following the suggestions [here](./misc/patches-rpip-30-suggestions.md)


### Changes since `v7`

The following updates have been made from [v7](./legacy/rewards-calculation-spec-v7.md) of the spec.


#### Major Updates

- Following the Protocol DAO vote to approve RPIP-30, [Collateral Rewards](#collateral-rewards) rewards are now weighted on a curve, favoring 8 Ether Minipools and creating diminishing returns as additional RPL above the minimum is staked.
  - This change is phased in linearly over 5 intervals, starting with interval 18, and will be fully effective in interval 23.
- There is no longer a maximum node collateral.


#### Minor Changes

- The Rewards File now includes a `totalNodeWeight` field which represents the sum of the rewards weighting.


#### Clarifications

---


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

If this value is 0, use the start time from the RPL token (also a Unix timestamp) as `startTime`:
```go
RocketTokenRPL.getInflationIntervalStartTime()
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

If the total amount of pending rewards is `0` for any reason, this interval cannot be used for rewards submission.
Simply wait for the protocol to roll over to the next interval and try again.

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

For each node, retrieve the **effective RPL stake**.
This should be calculated as follows.

For each minipool belonging to the node, get its current `state`:

```go
state := minipool.getStatus()
```

Ignore minipools that are not in the `staking` state.

Define `eligibleBorrowedEth` as the total amount of ETH borrowed by the node operator from the staking pool for eligible minipools.
Define `eligibleBondedEth` as the total amount of ETH the node operator has bonded for its eligible minipools.
Start with both set to `0`.

For each `staking` minipool, check if it was not exited from the Beacon Chain at the end of the interval:

1. Get the `status` of the validator from the Beacon Chain for `targetBcSlot` (e.g., `/eth/v1/beacon/states/<targetBcSlot>/validators?id=0x<pubkey>`). If the validator did not exist at `targetBcSlot`, ignore it and continue.
2. Get the `exit_epoch` for the validator.
3. If the validator's `exit_epoch` is **after** `targetBcSlot`'s epoch (`exit_epoch` > `targetSlotEpoch`), it is eligible.
   1. Add the amount of ETH borrowed by the node operator for this minipool to `eligibleBorrowedEth`:
        ```go
        borrowedEth := minipool.getUserDepositBalance()
        eligibleBorrowedEth += borrowedEth
        ```
   2. Add the amount of ETH bonded by the node operator for this minipool to `eligibleBondedEth`:
        ```go
        bondedEth := minipool.getNodeDepositBalance()
        eligibleBondedEth += bondedEth
        ```

Next, calculate the minimum and maximum RPL collateral levels based on the ETH/RPL ratio reported by the protocol:
```go
ratio := RocketNetworkPrices.getRPLPrice()
minCollateralFraction := RocketDAOProtocolSettingsNode.getMinimumPerMinipoolStake() // e.g., 10% in wei
maxCollateralFraction := RocketDAOProtocolSettingsNode.getMaximumPerMinipoolStake() // e.g., 150% in wei
minCollateral := eligibleBorrowedEth * minCollateralFraction / ratio
maxCollateral := eligibleBondedEth * maxCollateralFraction / ratio
``` 

Note that `minCollateral` is based on the amount *borrowed*, and `maxCollateral` is based on the amount *bonded*.

Now, calculate the node's effective RPL stake (`nodeEffectiveStake`) and weight (`nodeWeight`) based on the above:

```go
nodeStake := RocketNodeStaking.getNodeRPLStake(nodeAddress)
if nodeStake < minCollateral {
    nodeEffectiveStake := 0
    nodeWeight := 0
} else {
    if nodeStake > maxCollateral {
        nodeEffectiveStake := maxCollateral
    } else {
        nodeEffectiveStake := nodeStake
    }
    nodeWeight := getNodeWeight(eligibleBorrowedEth, nodeStake, ratio)
}
```

`getNodeWeight()` is defined in the [getNodeWeight section](#getnodeweight).

Next, scale the `nodeEffectiveStake` and `nodeWeight` by how long the node has been registered.
This prorates RPL rewards for new nodes that haven't been active for a full rewards interval, so they only receive a corresponding fraction of the rewards based on how long they've been registered.

For example, if the rewards period were 6300 Epochs (28 days) and a node registered 10 days ago, their `nodeEffectiveStake` and `nodeWeight` would be reduced to **35.7%** (10 / 28) of its true value.

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
    nodeWeight = nodeWeight * nodeAge / intervalTime
}
```

Finally:
- Sum each `nodeEffectiveStake` to retrieve the `totalEffectiveRplStake` across the entire network.
- Sum each `nodeWeight` to retrieve the `totalNodeWeight` across the entire network.

If the `totalEffectiveRplStake` or `totalNodeWeight` is `0` (i.e., *none* of the nodes are eligible for RPL rewards), add `collateralRewards` to `pDaoRewards`. Otherwise, if *any* node is eligible for rewards, perform the following steps instead.

Now, calculate the the cycle factor, `C`, of RPIP-30's phase-in. Define C to be on the closed range `[1, 6]` and calculate it:

```go
C := min(6, interval - 18 + 1)
```

You can now calculate the **collateral RPL per node** from `nodeWeight`, `totalNodeWeight`, `nodeEffectiveStake`, `totalEffectiveRplStake`, `C`, and `collateralRewards`.

```go
nodeCollateralAmount := (collateralRewards * C * nodeWeight / (totalNodeWeight * 6)) + (collateralRewards * (6 - C) * nodeEffectiveStake / (totalEffectiveRplStake * 6))
```

Sum the `nodeCollateralAmount` for each node to arrive at the `totalCalculatedCollateralRewards`.
As a sanity check, compare this to the original `collateralRewards` value using either the **total number of nodes** or the **total number of minipools**, whichever is higher, as a delta value:

```go
epsilon := max(numberOfNodes, numberOfMinipools)
if collateralRewards - totalCalculatedCollateralRewards > epsilon {
    // Raise an error because the calculation has excessive error
}
```

#### getNodeWeight

Calculate `stakedRplValueInEth`:

```go
stakedRplValueInEth = nodeStake * ratio / 1 Eth.
```

Calculate `percentOfBorrowedEth`:

```go
percentOfBorrowedEth = stakedRplValueInEth * 100 Eth / eligibleBorrowedEth
```

If `percentOfBorrowedEth <= 15 Eth`, return `100 * stakedRplValueInEth`.

Otherwise, return `((13.6137 Eth + 2 * ln(percentOfBorrowedEth - 13 Eth)) * eligibleBorrowedEth) / 1 Eth`.

`ln` is specified in the next section.

#### ln

`ln(x)` is defined as `log2(x) * 1 Eth / 1442695040888963407`

`log2` is defined in the next section.

#### log2

`log2(x)` is approximated using [iterative approximation](https://en.wikipedia.org/wiki/Binary_logarithm#Iterative_approximation) for 60 inner loops.

Define `result = 0`

Calculate `exponent` of the highest power of two that, when multiplied by 1 Eth, is less than or equal to the input `x`.
That is, if `x` is `40 Eth`, the highest power of two that is less than 40 Eth is 32 Eth, and the most significant bit of 0b0100000 (32 in binary) is at index 5 (counting the least signifigant bit as index 0), so `exponent` is 5.

Multiply the `exponent` by 1 Eth and add it to the `result`:

```go
result = result + exponent * 1 Eth
```

Next, calculate the iterative approximation's `y` term:
```go
y = x >> exponent
```

If `y` is 1 Eth, return `result`.

Otherwise, define `delta = 1 Eth`.

Loop 60 times. In each loop:
1. Divide delta by 2, `delta = delta / 2`
1. Square y, `y = y * y /  1 Eth`
1. If `y >= 2 Eth`:
    * Add `delta` to `result`, i.e. `result = result + delta`
    * Divide `y` by 2, i.e. `y = y / 2` 

After 60 loops, return `result`.

### Oracle DAO Rewards

Start by acquiring the list of Oracle DAO node addresses using the following contract methods:

```go

oDaoCount := RocketDAONodeTrusted.getMemberCount()
oDaoAddresses := address[oDaoCount]
for i = 0; i < oDaoCount; i++ {
    oDaoAddresses[i] = RocketDAONodeTrusted.getMemberAt(i)
}
```

Next, scale the amount of RPL earned by each Oracle DAO node by how long the node has been part of the Oracle DAO.
This prorates RPL rewards for new nodes that haven't been a member for a full rewards interval, so they only receive a corresponding fraction of the rewards based on how long they've been a part of the DAO.

For example, if the rewards period were 6300 Epochs (28 days) and a node joined 10 days ago, their share would be reduced to **35.7%** (10 / 28) of its true value.

The node's join time can be retrieved with the following contract method:

```go
joinTime := RocketDAONodeTrusted.getMemberJoinedTime(nodeAddress)
```

This should be subtracted from the timestamp of `targetElBlock` to determine the time since joining.
It should then be compared to `intervalTime` to determine the prorated rewards.

One way to accomplish this is to use the **number of seconds** the node participated in the interval as an analog to the "effective stake" in the Collateral RPL calculation above:

```go
odaoTime := targetElBlock.Timestamp - joinTime
participatedSeconds := intervalTime
if (odaoTime < intervalTime) {
    participatedSeconds = odaoTime
}
```

When finished, add each of these to retrieve the `totalParticipatedSeconds` for all oDAO nodes.

With this in hand, you can now calculate the **Oracle DAO RPL per node** by taking the original `oDaoRewards` value, multiplying by the `participatedSeconds`, and dividing by the `totalParticipatedSeconds`:

```go
oDaoAmount := oDaoRewards * participatedSeconds / totalParticipatedSeconds
```

Sum the `oDaoAmount` for each node to arrive at the `totalCalculatedODaoRewards`.
As a sanity check, compare this to the original `oDaoRewards` value using either the **total number of nodes** or the **total number of minipools**, whichever is higher, as a delta value:

```go
epsilon := max(numberOfNodes, numberOfMinipools)
if oDaoRewards - totalCalculatedODaoRewards > epsilon {
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


### Node Eligibility

For each registered node (the gathering of which was shown previously in the RPL calculation), observe the status and penalty count on each of its minipools:

Next, look at the minipools for the node with the following contract methods:
```go
nodeMinipoolCount := RocketMinipoolManager.getNodeMinipoolCount(nodeAddress)
nodeMinipools := address[nodeMinipoolCount]
for i := 0; i < nodeMinipoolCount; i++ {
    minipool := RocketMinipoolManager.getNodeMinipoolAt(nodeAddress, i)
    state := minipool.getStatus()
    penaltyCount := RocketNetworkPenalties.getPenaltyCount(minipoolAddress)
}
```

If the `state` is `staking` and the `penaltyCount` is **3 or more**, this node is a cheater and is not eligible for Smoothing Pool rewards.
Remove it from the list of eligible nodes and ignore it.

If the node has **at least one `staking` minipool**, then it is eligible for calculation. Otherwise, remove it from the list of eligible nodes and ignore it.


### (Optional) Minipool Eligibility

In addition to filtering out ineligible nodes, minipools can also be filtered to speed up calculations.
This is done by removing minipools that exited **before** the interval starts, or are scheduled to activate **after** the interval ends.

For each `staking` minipool in each eligible node, check the `activation_epoch` and `exit_epoch` for that minipool's validator:

1. Get the `status` of the validator from the Beacon Chain for `targetBcSlot` (e.g., `/eth/v1/beacon/states/<targetBcSlot>/validators?id=0x<pubkey>`).
   1. If the validator does not exist at that slot (its status is empty), or if its status is `pending_initialized` or `pending_queued`, it is not eligible for any rewards. Ignore it in the following calculations.
2. If the validator's `activation_epoch` is *after* `targetBcSlot`, it is not eligible. Remove it.
3. If the validator's `exit_epoch` is *before* `bnStartBlock`, it is not eligible. Remove it.


### Node Opt-In / Out Timing

For each eligible node, determine the **opt-in time** and **opt-out time**.
These will be used during attestation performance to determine if a given attestation should count towards the Smoothing Pool rewards or not.

Start by retreiving the opt-in status and the last time of status change for the node:
```go
isOptedIn := RocketNodeManager.getSmoothingPoolRegistrationState(nodeAddress)
statusChangeTime := rocketNodeManager.getSmoothingPoolRegistrationChanged(nodeAddress) // The contracts provide the Unix timestamp, in seconds
```

Use these details to determine the opt-in and opt-out time; for example:

```go
farPastTime := 0 // Some arbitrary timestamp that occurred before the start of the interval; the Unix epoch is fine for this
farFutureTime := 1e18 // Some arbitrary timestamp that will occur far after the end of the interval

if isOptedIn {
    optInTime = statusChangeTime
    optOutTime = farFutureTime
} else {
    optInTime = farPastTime
    optOutTime = statusChangeTime
}
```


### Calculating Attestation Performance and Minipool Scores

Start by defining the following variables:
- `totalMinipoolScore`, which cumulatively tracks the aggregated minpool scores for each attestation, starting at `0`
- `successfulAttestations` which tracks the number of successful attestations that were eligible for Smoothing Pool rewards, starting at `0`
- `minipoolScores`, a map of minipools to their individual cumulative minipool scores 

For each eligible minipool in each eligible node, make a note of its status and status change time:

```go
status := minipool.getStatus()
statusTime := minipool.getStatusTime()
```

For duties to be eligible for rewards inclusion, the minipool must be in the `staking` status at the time of the attestation duty assignment.
You may use the state of the chain at the time of the duty assignment or any state after the duty assignment to assess this.
This is used because `status` is one of the final states of a minipool (the other being `dissolved`, which is mutually exclusive with `staking`) and `statusTime` indicates the time at which the minipool entered `staking` status.
Thus, if a minipool's status is `staking`, it will always be `staking` and you can determine when it entered that state by using `statusTime`.

*Note that the `finalized` flag is not a true state and does not overwrite `staking`; it is a separate boolean value.*

Next, process the attestation performance of the minipool to gauge its `minipoolScore`.
Attestation performance is calculated on an Epoch-by-Epoch basis, from the first Epoch to the last Epoch of the interval, as follows for each Epoch:

1. Get the **attestation committees** for the Epoch (e.g., `/eth/v1/beacon/states/head/committees?epoch=<epochIndex>`)
2. Traverse the list of slots and committees, noting the `slotIndex`, `committeeIndex`, and `position` of an attestation assignment for the minipool (where `position` is the 0-based index of the entry in the response's list of validator indices). Ignore validators that do not correspond to *eligible* Rocket Pool minipools.
3. Get the block at `slotIndex` (e.g., `/eth/v2/beacon/blocks/<slotIndex>`).
4. Get the time of the block:
    ```go
    blockTime := genesisTime + secondsPerSlot * slotIndex
    ```
5. For the minipool corresponding to `position`:
   1. If `blockTime` occurred *before* the parent node's `optInTime` or *after* the parent node's `optOutTime`, this attestation is not eligible for Smoothing Pool rewards. Ignore it.
   2. If the minipool is not in `staking` status by the time of this attestation, it has not performed any eligible attestations yet so this duty should be ignored.
   3. If `blockTime` occurred *before* the minipool's `statusTime`, it was not in `staking` status during the attestation duty so this duty should be ignored.
      1. *Note that this check will only be relevant for solo staker migrations, as conventionally-created minipools will enter `staking` long before they begin attesting whereas solo staker migrations will be attesting prior to entering `staking` status.*
6. Look at the attestations in the subsequent blocks with matching `slotIndex`, `committeeIndex`, and `position`. Start at the block directly after `slotIndex`, and look up to 1 Epoch away (`BeaconConfig.SlotsPerEpoch`) from `slotIndex`.
   1. If one was recorded in *any of these blocks*, this attestation was successful. Calculate the `minipoolScore` for this attestation as described below.
   2. If the attestation was not found, it was missed. Add it to a running list of `missedAttestations`.
   3. The boundary is inclusive, so if an attestation for slot `n` is found in slot `n + BeaconConfig.SlotsPerEpoch` then it was successful. If it was found in slot `n + BeaconConfig.SlotsPerEpoch + 1`, it is too late and should be considered missed.

When a successful attestation is found, calculate the `minipoolScore` awarded to the minipool for that attestation:.

1. Add the attestation to a running list of `goodAttestations` for the minipool.
2. Get the amount of ETH bonded by the node operator and the commission (node fee) for this minipool on this block (the block corresponding to the attestation duty assignment) by using the block's timestamp and the timestamp of the minipool's last bond reduction:
    ```go
    currentBond := minipool.getNodeDepositBalance()
    currentFee := minipool.getNodeFee()
    previousBond := RocketMinipoolBondReducer.getLastBondReductionPrevValue(minipool.Address)
    previousFee := RocketMinipoolBondReducer.getLastBondReductionPrevNodeFee(minipool.Address)
    lastReduceTime := RocketMinipoolBondReducer.getLastBondReductionTime(minipool.Address)

    fee := currentFee
    bond := currentBond
    if lastReduceTime > 0 && lastReduceTime > blockTime {
         // If this block occurred before the bond was reduced, use the old values
        bond = previousBond
        fee = previousFee
    }
    ```
3. Calculate the `minipoolScore` using the minipool's bond amount and node fee:
    ```go
    minipoolScore := (1e18 - fee) * bond / 32e18 + fee // The "ideal" fractional amount of ETH awarded to the NO for this attestation, out of 1
    ```
4. Add `minipoolScore` to the minipool's running total, and the cumulative total for all minipools:
    ```go
    minipoolScores[minipool.Address] += minipoolScore
    totalMinipoolScore += minipoolScore
    successfulAttestations++
    ``` 

### (Optional) Removing Idle Minipools

Once each minipool's score has been determined, you may optionally remove minipools with zero score (i.e., no successful or missed attestations).
These minipools are considered idle and will just waste calculations in the final tally.

While skipping this step won't affect the final calculation, if you are logging the records of your calculation (such as with the "minipool performance" file included in the canonical Rocket Pool rewards intervals) then idle minipools may be omitted from the final report. 


### Calculating Node Rewards

Start by calculating the "ideal" amount of ETH that would go to node operators (normalizing `successfulAttestations` into an ETH amount in the process), based on their cumulative fractional scores:
```go
totalNodeOpShare := smoothingPoolBalance * totalMinipoolScore / (successfulAttestations * 1e18)
totalEthForMinipools := 0
```

Here we also define a variable `totalEthForMinipools` that will contain the cumulative total ("actual") amount of rewards for all node operators, which is initialized to 0.

Next, for each minipool, calculate the minipool's share of this ideal ETH total and add it to the cumulative total:
```go
minipoolEth := totalNodeOpShare * minipoolScores[minipool.Address] / totalMinipoolScore
nodeEth[minipool.OwningNode] += minipoolEth
totalEthForMinipools += minipoolEth
```
where `nodeEth` is the true amount of ETH awarded to each node.

Now, calculate the final "actual" pool staker balance (which will act as a buffer and capture any lost minipool ETH due to integer division):
```go
poolStakerEth := smoothingPoolBalance - totalEthForMinipools
```


## Constructing the Tree

With all of the above values, you can now create the Merkle tree for this interval using the [tree specification](./merkle-tree-spec.md).
