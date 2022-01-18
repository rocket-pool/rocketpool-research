# The Merge

After the merge, PoW will be abandoned and fees that were previously paid to PoW miners will instead go to the PoS validators who propose blocks. Validators will have the ability to configure the “coinbase” (called fee recipient post merge) of execution layer blocks which determines where the reward goes. 

In its current state, Rocket Pool has no way to enforce that the fee recipient field is set to the minipool address. And so, it is possible for validators to take 100% of the priority fees instead of sharing them with the rETH pool from which half of their validator stake was borrowed.

## Solution A - Penalty System w/ Rewards Sent to Minipool

Anticipating the merge, we included a “penalty” function within the smart contracts which, when enabled, would give the oDAO the power to impose penalties for node operators who do not comply with our requirement to share priority fees fairly. 

The penalty is defined as a percentage and is limited to a maximum value. Because of the uncertainty of MEV, the final max penalty value has left to a later date to be defined. The guardian can set it to any value and once we have determined what that value should be, the guardian role will be locked to prevent future changes.

In practice, it is expected that the penalty is never used. It’s mere existence should be enough to prevent node operators from taking priority fees for themselves.

### Contract Changes

Although we included the penalty feature, we purposefully left it as a bare minimum implementation to support the merge. We had to include it on launch because it needed to be built into the minipool delegate contract or we would run into a situation where people could refuse to upgrade their delegate contract and avoid the penalty process.

Only a single new contract needs to be introduced to support the merge. This new contract will be very similar to RocketNetworkBalances and RocketNetworkFees. It will allow oDAO members to vote on setting the penalty rate for individual minipools. Once majority (or super majority?) members vote on setting the penalty rate for a given minipool, setPenaltyRate will be called on RocketMinipoolPenalty which will record the rate for that minipool. On withdrawal, minipools already query this rate and withhold this percentage of the final balance from the node operator, sending it to the rETH users instead.

### Set Max Penalty Rate

The max penalty rate’s initial value is 0, resulting in the functionality being disabled by default. Guardian will need to call `setMaxPenaltyRate` on `RocketMinipoolPenalty` to set the rate to a value we calculate to be high enough to cover non-compliance of priority fee distribution.

### Smartnode Changes

#### Client Updates

There will be mandatory client updates to support the merge which may require additional configuration changes for the Smartnode stack to work correctly after the merge. At the very least, Smartnode will need to set the fee recipient address on the execution payloads to the validators minipool address. 

We are tracking the progress of https://github.com/ethereum/beacon-APIs/pull/178 which is a spec change that will introduce a new method to BN called `prepare_beacon_proposer`, this method will allow VC to advise the BN what fee recipient to use when it requests execution payloads from the EL client. 

We are also tracking the following client issues to ensure our supported clients allow for per validator fee recipients which would be a required feature if we choose this solution:

- https://github.com/prysmaticlabs/prysm/issues/9757
- https://github.com/status-im/nimbus-eth2/issues/3291
- https://github.com/sigp/lighthouse/issues/2883
- https://github.com/ConsenSys/teku/issues/4850

#### New oDAO Duty

The oDAO watchtower needs to be upgraded to include a new oDAO duty. The duty will be as follows:

1.	Monitor all new beaconchain blocks for blocks proposed by a Rocket Pool validator.
2.	If found, query the fee recipient value of the execution payload.
3.	Compare the fee recipient value with the withdrawal credentials of that validator (withdrawal credential will be the minpool's address).
4.	If a non-matching fee recipient is found, submit a vote to the new RocketNetworkPenalties contract to penalise the minipool.

### Notes

1.	What is a sufficient value for the max penalty rate? This doesn’t have to be perfect immediately as we can continue to adjust it until we transfer guardianship to a dead address.
2.	What is the function for determining the penalty rate to apply to a non-complying validator? Some possible options are:
a.	A set percentage for each non-complying block.
b.	An amount that is variable and calculated based on how much priority fee was withheld from rETH users.
c.	A percentage that increases with each non-compliance. (e.g. 1% then 5% then 10%)
3.	Once MEV becomes accessible to validators, the above values may need to be reconsidered.
4. Our minipool delegate is not setup to distribute rewards until the minipool is withdrawn so this solution locks rewards in the pool which is not efficient.
5. Consensus clients must add support for per validator account fee recipients for this solution to work.
6. This solution fails once rewards of a single validator exceeds 16 ETH. At this point, in the current design, anyone could call distribute on the minipool. The current design expects any balance over 16 ETH to be the final withdrawal amount and so that 16 ETH will be sent to rETH users and the NO will be slashed.

## Solution B - Per Minipool Reward Distributor Contract

The major drawback to solution A is that priority fees are sent to the minipool contract and are not accessible until after withdrawals are enabled. A more ideal situation would be for that ETH to be immediately available. This allows the ETH to enter the deposit pool and be used again in another validator resulting in more productive ETH in the protocol.

This solution would require validators to set their fee recipient to a secondary smart contract address which distributes the rETH users' share to the deposit pool and the node operators share to their withdrawal address.

The way this would work is that we use CREATE2 to determine what address this distributor contract will live at for each minipool. The deterministic address will be derived from the minipool address so it is in essence paired with the minipool. ETH can be sent to this address before the contract is even deployed so we can immediately enforce all validators to use these addresses as fee recipient. Penalties would apply for non-compliance just like in solution A.

Node operators at some point will execute some `initialiseFeeDistributor` method which would deploy the distributor contract at the predetermined address. Node operators could then execute `distribute` at whatever frequency they desire which would split the balance of that contract according to the minipool's nodeFee. 

### Contract Changes

Requires the same changes as solution A plus the following:

RocketMinipoolManager would have a new method `initialiseFeeDistributor` which can be called one time for each minipool. This could be done automatically for new minipools but would need to be called for ones created before this change. This method deploys a new RocketMinipoolFeeDistributor contract to a deterministic address.

RocketMinipoolFeeDistributor is a new contract which has a `distribute` method. Calling this method distributes the contract's balance to the deposit pool and the owner's withdrawal address, honouring the minipool's node fee.

The CREATE2 address is only deterministic if the contract code stays static, so we should utilise a proxy pattern and keep it simple so it never has to change. This is beneficial for gas costs anyway and would allow us to upgrade the distributor in the future by upgrading the delegate.

### Smartnode Changes

#### Client Updates

As with solution A plus the following:

A method for node operators to call `distribute`. Either an optional automatic operation, or a manual one.

#### New oDAO Duty

The oDAO watchtower is slightly modified from solution A. The duty will be as follows:

1. Monitor all new beaconchain blocks for blocks proposed by a Rocket Pool validator.
2. If found, query the fee recipient value of the execution payload.
3. Compare the fee recipient value with the deterministic "paired" address for that minipool.
4. If a non-matching fee recipient is found, submit a vote to the new RocketNetworkPenalties contract to penalise the minipool.

#### Balances Submissions

The watchtower would need to be modified to also count the ETH in the fee distributor contracts as part of the total ETH balance calculation.

### Notes

1. If the address of RocketMinipoolManager ever changes, so too will the distributor addresses. We might need to introduce one other "dumb" contract which performs the deploy and make it simple enough it will never be upgraded.
2. A downside is that node operators have to call distribute for the ETH to be sent to the deposit pool. If NO decides to sit on it for a long period, that's unproductive ETH in the protocol. Still preferable to solution A though.
3. This solution also requires clients to support a per validator fee recipient as well.
4. Gas is an ever-growing concern and this solution requires another contract deployment for each minipool plus a distribute transaction for each minipool.

## Solution C - Per Node Reward Distributor Contract

Deploying a new contract and calling distribute on each minipool will be a gas burden. Although it is expected that priority fees far outweigh the costs to deploy a proxy contract and call distribute, any wasted gas is wasteful on the protocol as it's a cost to the user to service the protocol. A more ideal solution would be to perform the distribution at a node level instead of at a per minipool level. Priority fees could accrue into a single contract which can then all be distributed with a single transaction.

For this to work, we would need to keep track of the node operator's average node fee. This would introduce a new storage variable on a node level which is the sum of all node fees. We can then calculate the average node fee by dividing it by the number of minipools they operate. Furthermore, any time the average node fee changes (e.g. new minipool created or minipool destroyed), `distribute` would need to be called to apply the distribution at that time based on the current average node fee.

### Contract Changes

Requires the same changes as solution A plus the following:

RocketNodeManager would have a new method `initialiseFeeDistributor` which can be called one time for each node operator. This could be done automatically for new nodes but would need to be called for ones created before this change. This method deploys a new RocketNodeFeeDistributor contract to a deterministic address. `initialiseFeeDistributor` will also iterate the node's minipools and sum the node fee into an average fee numerator storage value. This handles the migration of current node operators to this new system.

RocketNodeManager needs two new methods `increaseAverageFeeNumerator` and `decreaseAverageFeeNumerator`. These would increase and decrease a new storage value which keeps track of the sum of all the node operator's node fees. The average fee is calculated by taking this numerator and dividing by number of staking minipools. Both these methods should trigger a `distribute` on the distributor contract. Calling these methods if `initialiseFeeDistributor` has yet to be called will revert. This essentially blocks new minipools from staking until the node operator performs this required task.

RocketNodeFeeDistributor is a new contract which has a `distribute` method. Calling this method distributes the contract's balance to the deposit pool and the owner's withdrawal address, honouring the node's average node fee. The CREATE2 address is only deterministic if the contract code stays static, so we should utilise a proxy pattern and keep it simple so it never has to change. This is beneficial for gas costs anyway and would allow us to upgrade the distributor in the future by upgrading the delegate.

RocketMinipoolManager needs to call `increaseAverageFeeNumerator` on `incrementNodeStakingMinipoolCount` and `decreaseAverageFeeNumerator` on `decrementNodeStakingMinipoolCount`.

As a precaution, we could also update `RocketMinipoolManager` to revert on minipool creation if `initialiseFeeDistributor` has not been called. This prevents a minipool from being created but not able to stake due to not having performed this required function.

### Smartnode Changes

#### Client Updates

As with solution B.

#### New oDAO Duty

The oDAO watchtower is slightly modified from solution B. The duty will be as follows:

1.	Monitor all new beaconchain blocks for blocks proposed by a Rocket Pool validator.
2.	If found, query the fee recipient value of the execution payload.
3.	Compare the fee recipient value with the deterministic "paired" address for node that proposed this block.
4.	If a non-matching fee recipient is found, submit a vote to the new RocketNetworkPenalties contract to penalise the minipool.

#### Balances Submissions

The watchtower would need to be modified to also count the ETH in the fee distributor contracts as part of the total ETH balance calculation.

### Notes

- On weighing up the pros and cons of each solution, this solution is our preferred implementation as it results in immediate access to rewards as well as keeping gas burden low. We believe these two benefits outweigh the minor downside listed below.
- Because we are using average node fee instead of calculating the exact node fee on a per minipool basis, this solution acts as a bit of a smoothing pool on a node operator level. Over time, this should average out and not negatively affect the node operator's rewards. For example, if a NO has two minipools, one with 5% node fee and one with 20% node fee, if they receive a lot of rewards with the 20% validator, they would receive 12.5% fee on of those rewards instead of 20%. Conversely, if they received a lot of rewards on the 5% they would also receive 12.5% fee.
- This solution is technically more complicated and requires more contract upgrades than the others. But overall the UX is superior.
- MEV rewards would also be sent to this distributor contract. With the option to introduce a global smoothing pool at a later stage.
- This creates a clear separation between staking rewards and priority fees / MEV rewards and gives us more flexibility on handling withdrawals once the spec has been finalised.
- This solution does not require per validator account fee recipients as each node will have a single address. It is still desirable to have this option for users running an advanced setup that might have both Rocket Pool and non Rocket Pool validators.

# Withdrawals

Rocket Pool is currently designed to handle single withdrawals of all the rewards at the end of the validator's life. It is now becoming clear that this is an unlikely design outcome for the beaconchain withdrawal process. Skimming is currently being discussed and is very likely going to be a feature of Ethereum when withdrawals are enabled. We anticipate the chosen design to be such that balances greater than 32 ETH will have this excess balance deposited to the withdrawal address on some routine basis. Or some variation of this. The above solutions all handle priority fees (and eventually MEV rewards) but do no handle "skimming" staking rewards. 

This process will be distinct from priority fees and MEV rewards which do not have to go to the withdrawal address. It is also likely that a withdrawal receipt will be provided for each skim and the final withdrawal when exiting the beaconchain. This receipt will be a merkle proof which can be verified on the execution layer as the root hash will be committed to chain. This solution is highly desirable for Rocket Pool as it means we can upgrade our minipool contract to accept the merkle proof and distribute the rewards accordingly. This means the oDAO does not gatekeep withdrawals which is a strong design goal of Rocket Pool. Having priority fees and MEV rewards go to a second address also is desirable in this design as it keeps the balances accounted for with receipts and balances not accounted for with receipts separate. Meaning we can prevent any distribution of funds in the minipool address without a withdrawal receipt.

# Smoothing Pool

These solutions are stepping stones to the smoothing pool idea. Solution C is already somewhat of a smoothing pool just limited to each node. A global smoothing pool would be an address where all opted in node operators must send their MEV and priority fees. How the fees would be then distributed is a topic for another research doc.
