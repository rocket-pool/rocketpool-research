# Rocket Pool Incentivization Ideas

In this section, we discuss our incentivization ideas for attracting both node operators and normal stakers to Rocket Pool.

**NOTE: Many of these ideas assume the successful deployment of enforcement mechanisms within the protocol to ensure that validators "play by the rules" (by sharing their MEV and Priority Fee earnings with the rETH staking pool).
We suggest you put the notion of "cheating" aside while reading about these ideas, and consider them on their merits; that is, consider the benefit each idea might convey to Rocket Pool users without concern about securing against bad actors.
Once you're done reading, please review the details of our enforcement mechanism proposals in the [Security](./Security.md) section for more information.**


## The Smoothing Pool

In the [Analysis](./Analysis.md) section, we made it abundantly clear that MEV and priority fees are projected to make up a considerable amount of a node operator's income - in some cases, more than doubling proposal and attestation rewards on eth2.
While the Flashbots research paper suggests that MEV will unconditionally provide better returns than conventional node operation alone, it also indicates there is likely to be a **significant luck factor** involved in a node operator's MEV earnings.
Either by a shortage of block proposal opportunities, relatively low-reward bundles, or a period of low activity on the chain, the range of expected MEV rewards will likely vary greatly from the luckiest to the unluckiest case.

The eth1 chain experiences something similar with Proof of Work.
Proof of Work is an all-or-nothing game, and miners with comparably low hash-rates face a tremendous amount of uncertainly with respect to their rewards.
If they get lucky, they could find a block much sooner than expected and thus receive block rewards faster than the average miner.
If they get *unlucky*, they could go for a very long time without finding a block, and effectively never receive a payout for their work. 

The ubiquity of mining pools today strongly suggests that users prefer a **socialized reward structure**, where many operators aggregate their hash power together and distribute all of the total rewards evenly to the participants.
This socialization has the effect of **smoothing out the rewards** by transforming the high-risk, high-reward structure of solo-mining into a stream of more regular income with smaller, more frequent payouts.

The rETH staking pool currently enjoys this scheme, as its returns are averaged across all of the Rocket Pool node operators.
Node operators, however, don't have access to such a mechanism.  
To that end, Rocket Pool is investigating an optional feature that mimics this "smoothing" effect for node operators as well.
We call this new capability **the smoothing pool**. 

The smoothing pool is a smart contract that Rocket Pool node operators may **opt into** if they so desire.
If node operators opt into the smoothing pool, their MEV earnings (and possibly Priority Fees, depending on whether or not `0x02` is accepted) will be sent to this pool.
These pooled rewards will be **evenly distributed to all participating Rocket Pool node operators** at regular intervals.
This distribution could be done similarly to RPL rewards checkpoints (requiring node operators to "pull" their rewards), or done automatically via the Oracle DAO in a "push" configuration (predicated on gas fees, of course).
The exact mechanics of distribution will be determined at a later date if the idea gains significant traction within the community.

As expected, **a fair portion of these smoothed rewards will be shared with the rETH staking pool**.
The exact amount could be a monolithic setting controlled by the Protocol DAO, which may serve as an enticement to participate in the protocol's governance.
In this mode, we would need to select a value that balances the incentivization for both normal users to stake with Rocket Pool and for node operators to create Rocket Pool nodes.
A strong initial candidate involves a 45% allocation to the rETH pool and a 55% allocation to the node operators.
This mirrors the 10% average commission fee that we employ to incentivize operators to join Rocket Pool.
Alternatively, we could use the Oracle DAO to calculate the average commission across all participants of the pool, and use the resulting figure when calculating the rETH return amount. 

**If node operators opt out of the smoothing pool** (the default behavior), they will operate their validators in "solo" mode; their MEV rewards are simply sent to their **minipool** where it can be withdrawn at any time and fairly distributed between the node operator and the rETH staking pool.
They will be at the mercy of their own luck; however, they will be able to trigger a `payout` on their minipool and collect the rewards at any time (after they have been split between the operator and the rETH staking pool appropriately).

Switching between the two modes would be possible, though it would need to come with a requisite cooldown period to prevent gaming the system.

In either mode, **node operators would receive >100% return on their fair share of the MEV rewards, just as they do with eth2 rewards today**, thanks to the commission system.
We expect the smoothing pool to be a unique and popular capability that will encourage a fair distribution of rewards; in conjuction with the resulting variance reduction, we expect increased incentivization in using Rocket Pool for node operation.

Note that the smoothing pool is still just an idea at this point, and its implementation is largely undecided.
We want to gauge the community's feedback on this idea before pursuing its development.
If there is significant interest, we can explore it further in the time between Rocket Pool's launch on Mainnet and the Merge. 


## MEV Provider Partnerships

One of our main takeaways from the discussion surrounding `0x02` and Ethereum in general is that MEV is likely here to stay.
If it cannot be avoided, then it must be accepted; however, not all MEV is equal.

Rocket Pool believes in the support of transparent, traceable, democratized MEV systems that do not leverage "shadow" transaction pools or side channel payments.
To that end, we have begun to collaborate with block-searching and bundle-providing organizations with a proven track record of providing this variety of MEV.

Our first collaborator in this domain is [Flashbots](https://docs.flashbots.net/).

As part of this collaboration, we could start by adding MEV-Geth as an additional execution client option in our `service config` interview process (part of the initialization of a new Rocket Pool node.

Node operators that want to partake in MEV could use this option.
In doing so, we would give node operators the ability to automatically take advantage of democratized, transparent MEV without any extra setup or knowledge on their end.

The system would work as follows:

- A Rocket Pool validator is up for block proposal.
- Our partners would provide bundles to the validator.
- The MEV rewards and priority fees would be sent to one of two locations, based on the node operator's choice: the `coinbase` (which would be the minipool on eth1 for that validator), or the **smoothing pool**. 
  - If `0x02` is accepted, the priority fees for that block would be sent to the `coinbase` (the minipool), and the MEV rewards could either be sent there as well or to the smoothing pool contract. In the latter case, MEV would have to be delivered via a separate transaction included in the block instead of given to the `coinbase`. We would need to compensate the MEV provider for the gas associated with that transaction.
  - If `0x02` is **not** accepted, the `coinbase` itself could either be the minipool address or the smoothing pool address. Both MEV and priority fees would be sent to the same location. There would be no need for a separate transaction, though we would need to add some way to track which node operator was responsible for which deposit into the smoothing pool. The Oracle DAO could do this, but it would add more reliance upon it (which may want to be avoided in the name of decentralization).
- The node operator could then trigger a withdrawal of these funds from the minipool via the `payout` function, as described in the smoothing pool section, or simply let them accumulate.

By the sheer virtue of ease-of-use, along with access to these bundle providers' services, we anticipate that this will encourage Rocket Pool node usage for the majority of users who want turnkey access to competitive returns while simultaneously promoting the relative health of the network.

Furthermore, providing this capability (when coupled with the security measures that enforce fair distribution of funds) would make Rocket Pool's rETH returns quite competitive as demonstrated in the [Analysis](./Analysis.md) section.
We expect that these collaborations would have the largest effect on the overall longevity of the protocol, and we are closely exploring them.


### The EV Leaderboard

We are exploring plans to gamify the MEV rewards earned by nodes.
As an example, Rocket Pool could keep a running **leaderboard** of the minipools that have contributed the most ETH back to the rETH pool.
At the end of each checkpoint, the top pools would receive prizes in RPL as a reward for demonstrably contributing to the health of the protocol.
A fraction of the total annual RPL inflation would be diverted to this prize pool.

This would be analogous to the way in which we provided rewards during our beta tests, where the nodes that accumulated the most ETH on the Beacon Chain during the beta period were rewarded with RPL proportionally.
Unlike the beta rewards, this would apply to MEV earnings and Priority Fees - **not Beacon Chain rewards** for block proposal and attestation.

The EV Leaderboard could be reset at every checkpoint, giving everyone an equal chance, or it could persist across multiple checkpoints.

We are curious to know if the community would be interested in such a system, which - again - would be a unique incentive that Rocket Pool could provide to its node operators.
