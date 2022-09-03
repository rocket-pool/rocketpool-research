# [Draft] Specification for the Penalty System

**This document is still a DRAFT and is subject to change until the release of Rocket Pool Redstone on Mainnet.**

## Background

Rocket Pool is a trustless, permissionless protocol.
Anyone can join as a node operator as long as they put their fair share of collateral up for each validator.

When the protocol was originally conceived, the understanding was that all of a validator's rewards would be provided to its address on the Beacon Chain.
All of those rewards would ultimately be transferred to the validator's withdrawal credentials (in our case, the minipool contract on the Execution layer).
As the minipool contract enforces the fair reception and distribution of those rewards, there was no special action necessary to assess said distribution.

However, during the middle of 2021, it became clear that this was not going to be the case.

The advent of the "Quick Merge" indicated that a validator's rewards would actually be split into two categories, and live on two separate chains:

- Attestation rewards are provided on the Beacon Chain
- Block Proposal rewards are provided on the Beacon Chain
- Sync committee rewards are provided on the Beacon Chain

- Priority fees (transaction tips given during block proposals) are provided **to an address of the Node Operator's choosing on the Execution Layer**
- MEV extraction is provided **to an address of the Node Operator's choosing on the Execution Layer**

The complication here is that priority fees and MEV are both sent to an arbitrary Execution layer address that is completely up to the discretion of the Node Operator.
Neither the Rocket Pool protocol, nor the general Ethereum core protocol, have a way to enforce its assignment while permitting trustless, permissionless node operation.

As documented extensively in [our attempt to (partially) resolve this issue](https://github.com/ethereum/consensus-specs/pull/2454), the issue here is that a malicious Node Operator would be able to adjust this address to one that they control, thus pocketing the priority fees and MEV without needing to share them amongst the staking pool.

It became clear during this research period that the core protocol was evolving in such a way that it would not be able to enforce the fair sharing of these rewards, and thus Rocket Pool required an additional layer on top of the core protocol that could satisfy this need.
From this requirement, the **Penalty System** was born.


## The Penalty System

The penalty system is a function built into the current minipool contract for all of Rocket Pool's minipools.
The rules it follows are simple:

- Each minipool has its own **penalty counter**. This is a `uint256` value stored in the `RocketStorage` contract that tracks the number of penalties applied to that minipool.
  - The value can be retrieved via this contract function:
    ```go
    penaltyCount := RocketStorage.getUint(keccak256("network.penalties.penalty", minipoolAddress))
    ```
- When a malicious condition is detected by the Oracle DAO, the Oracle DAO can vote to issue that minipool a **penalty**. This will increase the penalty counter on that minipool by 1.
  - A voting quorum of 51% of the Oracle DAO members is required.
  - The malicious conditions mentioned are discussed in a subsequent section.
- The **first two penalties** are known as **strikes**. These apply as "warnings", indicating that a malicious condition was detected but no actual penalty has been applied to the minipool. The intent is to be lenient on node operators who accidentally misconfigured their system, and prompt them to correct it without any action being taken.
- The **third penalty and onward** are known as **infractions**. Each infraction will result in **10% of the node operator's share of the Beacon chain balance** distributed by the Ethereum protocol (either partially, during skimming, or fully, during a validator exit) to be rerouted to the staking pool instead.
  - This applies to both the rewards **and the initial 16 ETH deposit**.
  - The maximum penalty that can be levied on a minipool is **80%**, which still leaves some incentive to exit the validator.

For example:
  - If a node operator exits with one infraction, they will receive 90% of their expected ETH from the Beacon chain.
  - If a node operator exits with five infractions, they will receive 50% of their expected ETH.
  - If a node operator exits with eight or more infractions, they will receive 20% of their expected ETH.


## Penalizable Conditions

The following is a list of conditions the Oracle DAO can detect to issue a penalty on a minipool.

**NOTE: for all of the below conditions, the address `0x000...000` is *not* a legal address, because the proposal resulted in the loss of ETH for the pool stakers. Proposals with a fee recipient of this address will be assigned a penalty.**


### Users that Never Joined the Smoothing Pool

If the user *never* joined the Smoothing Pool, they can have the following **legal** fee recipients:
- The rETH contract address
- The Smoothing Pool contract address
- The node's **fee distributor** contract address

If the minipool proposes a block with a fee recipient other than *any one of these three addresses*, the minipool will be issued a penalty.


### Users that are Actively Enrolled the Smoothing Pool

If the user is **opted into** the Smoothing Pool at the time of the block proposal, they can have the following **legal** fee recipients:
- The Smoothing Pool contract address

If the minipool proposes a block with any fee recipient *other than the Smoothing Pool address*, the minipool will be issued a penalty.


### Users that Recently Unenrolled from the Smoothing Pool

If a user was previously opted into the Smoothing Pool and opts out, their fee recipient **must remain as the Smoothing Pool address** until the Epoch *after* they opted out has been **finalized**.

For example:

- Bob opts into the Smoothing Pool on slot 30 (Epoch 0).
- Some time later, Bob opts out of the Smoothing Pool on slot 32005 (Epoch 1000).
- Bob queries his Beacon node, and receives confirmation that Epoch **1001** (1 after the Epoch he opted out in) was finalized by slot 32064 (Epoch 1002). After slot 32064, Bob can now safely change his fee recipient to his node's fee distributor contract address.

The rationale here is to prevent people from noticing they have an upcoming proposal assignment and intentionally front-running it with an opt-out transaction so they collect the priority fees and MEV for themselves.
As proposal assignments are only fully decided the Epoch **before** the one in which the proposal is due, this "cooldown" period lasts until the Epoch after the opt-out transaction has been finalized to ensure the user could not have taken advantage of this lookahead functionality.