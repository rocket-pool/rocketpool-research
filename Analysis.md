# Post-Merge APY Analysis

After the merge and transition to Proof-of-Stake, Ethereum validators will earn rewards from four distinct paths:

- Attestation rewards for successfully attesting to proposed blocks (**sent to the validator address on the Beacon Chain**)
- Block proposal rewards for successfully proposing a block (**sent to the validator address on the Beacon Chain**)
- Priority fees (formerly transaction fees) (**sent to the proposed block's `coinbase` address on eth1**)
- [Maximum Extracted Value (MEV)](https://ethereum.stackexchange.com/questions/94613/what-is-miner-extracted-value-mev) (**sent to an arbitrary address on eth1**)

As part of the conversation surrounding the Merge, there has been much speculation regarding the impacts of the latter two portions on a validator's annual returns.
Many discussions tend to target best-case or worst-case scenarios, which are useful in thought experiments but need to be analyzed in terms of practical numbers to assess their true merit.
Unfortunately, reliable numbers for priority fees and MEV extraction in a Proof of Stake world are not currently possible to derive due to the unknown impacts of EIP 1559 and Layer 2 solutions.
The best we as a community can do is estimate based on existing data.

As part of our exploration into the consequences of priority fees and MEV on Rocket Pool, we have constructed a model that includes all four reward paths.
Our analysis is based on this model.
These numbers are meant to serve as **rough estimates only** and will almost certainly deviate significantly from real-world conditions after the merge, but the raw numbers themselves are not particularly important; rather, we use this model to assess the **relative differences** across various scenarios.
If the numbers are inaccurate, they should impact each scenario in the same way.
This means that the relative differences are preserved.

Our model is available in this repository as `FeeMEVAnalysis.xlsx`.
Below is a walkthrough of the context surrounding the model.


## Variables

The following variables are used in our model:

- **Slots per year**: We use **2,629,746** for the expected number of slots (blocks) per year. This is derived from the fact that the Beacon Chain expects a slot every **12 seconds**. To reach this, we state that a year is **365.2425 days** long.
- **Staked ETH**: The amount of ETH staked by validators.
- **Issuance ETH**: The amount of ETH generated per year from attestation and block proposal rewards (taken from [Justin Drake's model](https://docs.google.com/spreadsheets/d/1FslqTnECKvi7_l4x6lbyRhNtzW9f6CVEzwDf04zprfA/edit?usp=sharing))
- **Number of Validators**: Directly derived from the total staked ETH.
- **Gas Price in gwei**: The average gas price to use when calculating the impact of priority fees.
- **Daily Fees in ETH**: The amount of ETH provided from priority fees per day, using the provided gas price (estimated using a 15,000,000 gas limit per block).
- **Burn Rate**: The amount of the priority fees that are burned after EIP 1559 has been introduced.
- **Fees per Year**: The per-year earnings for validators from the non-burned portion of priority fees.
- **Slots per Year**: The average number of slots (blocks) that a validator will be given, derived from the slots per year and the number of validators on the network.
- **Reward per Slot**: The average amount of ETH minted per block on the Beacon Chain - derived from the annual issuance divided by the number of slots per year.
- **Fees per Slot**: The average amount of priority fees that are transferred to a validator as part of a block proposal - derived from the fees generated per year, divided by the number of slots per year.
- **MEV per Slot**: An estimate for the average MEV earnings taken by a validator in a block proposal. These figures were extracted from [this recent study by Flashbots](https://hackmd.io/@flashbots/mev-in-eth2).
- **RP Node Commission**: The commission awarded to a Rocket Pool minipool for the sake of this analysis.
  

## Analysis Assumptions

In our analysis, we use the following figures:

- **6,000,000 staked ETH** which corresponds to **187,500 validators** on the network - a close approximate to the number of validators on mainnet today. For future projections, we have also included columns with higher amounts of ETH staked in the model.
- **20 gwei** for a gas price, which is a reasonable figure given the recent trends in the market.
- **0.1 ETH of MEV per slot**.
- A **10%** Rocket Pool commission.

Using these numbers, we obtain the following **baseline results for a solo staker**:

- 2.17 ETH per year from Beacon Chain rewards (block proposal and attestation rewards), which results in **6.78% APY** without priority fees or MEV included.
- 0.97 ETH per year from priority fees, which accounts for a **3.04% APY** from fees alone
- 1.40 ETH per year from MEV, which accounts for a **4.36% APY** from MEV

All told, this sums to 4.54 ETH per year on average, which corresponds to a **14.19% APY** for a solo staker.


## Baseline Scenarios

In Rocket Pool's case, a node operator only stakes 16 of their own ETH.
The current usage of the 0x01 credentials means that validator duty rewards are fairly socialized between the node operator and the rETH staking pool which contributed the supplemental 16 ETH, minus a small commission to incentivize the node operator.

If we assume that commission is the average of 10%, then a node operator can be expected to earn 55% of the typical validation-based rewards, which comes to 1.19 ETH per year.
However, this validator will still earn **all** of the MEV and priority fees, which were earlier estimated at 2.37 ETH.
This reveals that as-is today, a Rocket Pool node operator would earn 3.56 ETH in rewards per year, which is a **22.25% APY**.

These supplemental rewards don't come from nowhere; they are effectively stolen from the staking pool.
The staking pool would be left with 45% of the 2.17 ETH generated on eth2 from proposal and attestation rewards, which amounts to 0.98 ETH.
This is a reward of **6.10% APR**.

This can be seen in the **Not sharing fees or MEV** scenario in the model.

This indicates that without taking any action, other pools have the capability of offering a staking return rate up to 14.19% whereas Rocket Pool would only be able to offer 6.10% (in the worst case where *none* of the node operators share their priority fees or MEV earnings).
This discrepancy is the source of the problem revealed by the conversation surrounding the `0x02` proposal.

We have included another scenario in the model, titled **Sharing fees not MEV**, that reflects a scenario where `0x02` is accepted and priority fees are distributed fairly.
In the scenario, the node operator earns 3.12 ETH per year (a **19.52% APY**) while the rETH staking pool earns 1.31 ETH per year (an **8.84% APY**).
This is better than the previous scenario, but still falls short of competitive rates.
It also assumes that priority fees would not change in response to `0x02`, whereas the discussion of the proposal indicated that it would encourage side-channel payments and thus reduce the actual priority fees being distributed.
Nevertheless, this scenario is included as a point of reference.

A scenario in which *everyone* shared their rewards appropriately would result in 2.50 ETH per year for node operators (a **15.60% APY**) and 2.04 ETH per year for the rETH staking pool (a **12.76% APY**).
This is shown in the **Sharing** scenario in the model.
It demonstrates that if socialization of all rewards can be enforced, then the pool is immediately competitive with other offerings which can offer a **maximum of 14.19%** (but will likely take a commission on MEV earnings and priority fees as well, which will lower this amount). 

These scenarios are summarized in the following table.

| Scenario | Node Operator Returns | rETH Pool Returns |
| - | - | - |
| Solo staking, normalized to<br/>16 ETH (for reference) | 2.27 ETH (**14.19% APY**) | 2.27 ETH (**14.19% APY**) |
| Sharing all returns | 2.50 ETH (**15.60% APY**) | 2.04 ETH  (**12.76% APY**) |
| Sharing priority fees but not MEV<br/>(requires `0x02` to be accepted) | 3.12 ETH (**19.52% APY**) | 1.31 ETH (**8.84% APY**) |
| Not sharing any rewards | 3.56 ETH (**22.25% APY**) | 0.98 ETH (**6.10% APR**) |

The team is currently exploring ways to incentivize fair sharing of these rewards with the rETH staking pool, and have suggested several options described in the [Incentivization](./Incentivization.md) and [Security](./Security.md) sections appropriately.
The relevant enforcement mechanism discussed here is the **slashing and negative commission** system.
In this system, when a "cheater" is detected, the node operator's RPL is auctioned and their commission is set to a negative amount.
The proceeds from the RPL auction, and a large a mount of the validator's Beacon Chain balance, are then provided to the rETH staking pool as compensation for the stolen funds. 

For more information on this system and its corresponding detection mechanisms, please visit the [Security](./Security.md) section.


## Scenarios Including Enforcement

This segment of the model demonstrates what happens to a node operator's returns in the event that they are detected during selfish activities and punished by the slashing and negative commission system.
They assume that **75% of the validator's balance** on the Beacon Chain will be refunded to the rETH staking pool (a commission rate of -75%).
This was chosen as an arbitrary but severe penalty because as the penalty increases, less total ETH is given to the node operator and thus they have less incentive to exit the validator.
In the worst case, they could hold the Beacon Chain balance "hostage" out of spite, ensuring that the rETH pool never receives its fair share from that validator.
This value could certainly be optimized in the future, and could even vary depending on the severity of the node operator's transgressions.

The model reveals the following scenarios (assuming there is **no compounding interest**, after **11 years of operation**):

| Scenario | Node Operator Returns | rETH Pool Returns |
| - | - | - |
| Sharing all returns | 27.45 ETH (**171.59%**) | 22.46 ETH (**140.39%**) |
| Sharing priority fees but not MEV,<br/>and slashed (requires `0x02` to be accepted) | 22.36 ETH (**139.76%**) | 27.55 ETH (**172.22%**) |
| Not sharing any rewards, and slashed | 27.17 ETH (**169.82%**) | 22.74 ETH (**142.16%**) |

This indicates that using the model's averaged projections for priority fees and MEV earnings, it will take **over 11 years** to break even and begin earning more via cheating than a node operator would via sharing.
It also indicates that `0x02`'s acceptance would offer a nontrivial security benefit to Rocket Pool in cases where priority fees are not completely overshadowed by MEV earnings, but is not explicitly required for Rocket Pool's success.

Adjusting the numbers for variance lowers this break-even point, but it is still a major disincentive to act selfishly.
For example, *quadrupling* the expected MEV rewards to 0.4 ETH per block will still result in a break-even period of **over 4 years** if a node operator pockets the priority fees and MEV earnings.


## Summary

This model demonstrates that with the estimates for MEV earnings and priority fees proposed by the Flashbots team and Justin Drake, Rocket Pool can be made quite competitive through the use of incentivization and protocol rule enforcement. 
That being said, we want to reiterate that **these figures are meant purely for relative comparison** of the provided scenarios.
They are not intended to be used as a reference for Rocket Pool's projected return rates.

We encourage the community to explore the model, improve it where possible, and use it to discuss the implications for Rocket Pool and the post-merge Ethereum chain as a whole.
