# Security and Protocol Health Enforcement

In this section we propose ideas for ways in which Rocket Pool can enforce the health of the protocol.
Our discussion primarily aims at enforcing a fair share of a portion of a node operator's MEV earnings and priority fees with the rETH staking pool, though other, related topics are also welcome in this section.


## The Challenge with Decentralized Staking Pools

In [PR #2454](https://github.com/ethereum/eth2.0-specs/pull/2454), we proposed a new withdrawal credential prefix - `0x02` - and sought its addition to the eth2.0 specification.
This prefix would be an extension of the existing `0x01` prefix (rather than specify a withdrawal key, it would hold an eth1 address which the validator rewards would be sent to upon a validator exit and withdrawal).
In addition, `0x02` would come with a condition related to the quick merge: blocks proposed by 0x02 validators would be invalid unless the `coinbase` for those blocks matched the withdrawal address.
In other words, under these credentials, the eth2.0 protocol would send priority fees to the validator's withdrawal address as well as block proposal and attestation rewards.

In spirit, `0x02` attempts to *enforce socialization of all block-proposal-related fees amongst the owners of the 32 ETH that created that validator*.
For Rocket Pool, this means that the rETH staking pool should receive half of all rewards associated with proposing a block (minus the minipool's commission).

The proposal has generated considerable discussion within the Ethereum R&D community.
While founded on good intentions and offering several beneficial effects, the proposal has also revealed some unexpected drawbacks related to the health of the Ethereum network.
Unfortunately, the proposed `0x02` mechanism may not accomplish its intended goal due to the concept of [Maximum Extracted Value (MEV)](https://ethereum.stackexchange.com/questions/94613/what-is-miner-extracted-value-mev).
MEV is a major obstacle in the fair implementation and usage of Ethereum in general - not just Rocket Pool.
Indeed, [several Ethereum researchers](https://www.coindesk.com/ethereum-mev-frontrunning-solutions) have [written about](https://github.com/tkstanczak/mev-geth) the issue already.
Simply put: if abused, MEV can [harm the platform](https://research.paradigm.xyz/MEV).

While `0x02` would socialize priority fees and MEV earnings delivered via the `coinbase` address, there are ways to earn MEV that bypass the `coinbase` address.
Specifically, `0x02` may encourage **side-channel payment setups**.
In theory, it would be more lucrative for a validator to set up a private channel with parties who offer them transactions with negligible gas costs, but provide the validator personally with a hefty payment in exchange for including the transactions in the next block. This actually *reduces* the amount of shareable priority fees, while *increasing* the amount of ETH captured selfishly by the node operator.

The Ethereum protocol itself currently does not have the ability to prevent this variety of side-channel activity which is largely considered to be unhealthy - so unhealthy, in fact, that its implication has been enough to dissuade the adoption of `0x02` into the eth2.0 specification.

**What this means for Rocket Pool**: a sufficiently determined node operator could create an infrastructure that includes these side channels, and thus capture a large portion of the total block proposal rewards without sharing them with the rETH staking pool.
As we discussed in the Analysis section, this would cause a tremendous increase in the node operator's APY and a correspondingly tremendous *decrease* in the rETH pool's APY (if this behavior is common enough to affect the total pool's return rate).
As a result, the rETH staking pool's returns would no longer be competitive compared to other (centralized) pools that don't have this problem, and the health of the protocol would be threatened.

While there is considerable research going into reducing MEV's practicality or potential across the network in future iterations of the Ethereum protocol, Rocket Pool must assume that MEV will be present in the quick merge and prepare accordingly.
In the following sections, we offer some ideas on ways in which we can enforce the intended *spirit* of `0x02`, and thus maintain the prosperity of the pool.

As an important note, **all of the ideas provided in this idea are preliminary**.
This description **does not constitute a commitment from Rocket Pool to employ them**; it is purely being offered as a set of ideas on how the drawbacks of `0x02` could be addressed.


## The Negative Commission Enforcement Mechanism

All of the ideas regarding deterring node operators from acting selfishly ultimately fail unless there is a *reason* for them not to act selfishly.
The easiest way to construct such a reason is to threaten to withhold portions of their staked RPL and staked ETH.
The former can be done by the Oracle DAO today, by voting to label the minipool as **slashed** on the Beacon Chain; the latter requires a new mechanism that we are exploring: the ability for the Oracle DAO to set the commission rate of a minipool **to a negative number**.
This would have the effect of redirecting a sizeable portion of the validator's Beacon Chain balance to the rETH staking pool, as a way to account for the "stolen" funds. 

There are a few drawbacks to this system:

- It would not adequately compensate the rETH staking pool if the "stolen" funds amounted to more than the validator's total balance on the Beacon Chain.
- It does not prevent future theft - doing so would require node operators to share their validator keys with the Oracle DAO, which would use them to force a validator exit upon confirmation of selfish activity. We have decided that this would against the decentralized spirit of Rocket Pool and are not pursuing this vector.

That being said, in the Analysis section we demonstrate that on average, it would take a selfish node operator **several years** to break even from losing their initial deposit to a combined slashing and negative commission setting.
We expect that this will act as enough of a deterrent to prevent selfish behavior, providing that behavior can be detected.

It would be trivial to check if a node operator has changed the `coinbase` argument to something other than the minipool address (or the smoothing pool address) and punish them accordingly.
We envision a **fraudproof system**, where anyone can submit an on-chain transaction documenting such an occurrence and flagging that node operator automatically.
We could even **offer an incentive to such a reporter**, much in the same way that the Beacon Chain offers incentives for reporting slashing offenses today.

Detecting selfish behavior done via side channels or custom transactions is a more difficult challenge.
We have several ideas for such detection, which are proposed below. 


## Option 1: The Side-Channel Monitoring System

This is a new system that Rocket Pool is experimenting with to deter side-channel activities, and ensure that blocks proposed by Rocket Pool node operators only contain transactions that were part of the public mempool, or came from a collaborating MEV provider (see the Incentivization section for more information on these).

In this system the Oracle DAO would actively and continuously monitor the public Ethereum mempool across hundreds of nodes distributed around the globe.
It would catalog each transaction that enters the pool, including the proposed gas cost, the time of entry, the sender and receiver, and the contents via the transaction's hash.
Each block proposed by a Rocket Pool minipool would have its contents cross-checked against these records.

Any transaction appearing in the block, but *not appearing* in the canonical mempool of any of the execution clients monitored by the Oracle Nodes (before a certain time threshold) and *not originating* from our MEV partners would be flagged as suspect, and the minipool would be issued a **strike**.
To mitigate false positives, the strike count on a minipool would periodically reset after a prolonged period of "good behavior", where the minipool proposes only "legal" transactions.

In other words, this system would "whitelist" transactions coming from the canonical Ethereum gossip channel and ones coming from our MEV provider partners.
**Flashbots already has a system quite similar to this in place today** for the purposes of monitoring the network; as part of our collaboration, we are looking to augment it for this particular use case.

If the validator consistently produces blocks with transactions not present on the canonical Ethereum gossip channel, or includes transactions with anomalously low gas prices relative to the other transactions, it is likely that the operator is engaging in side-channel activity or unsanctioned MEV which does not contribute to the staking pool.
Once a minipool has accumulated enough strikes, the Oracle DAO would trigger a **slashing** and **negative commission** on the minipool, and all other minipools run by that node operator.

The exact mechanism of this system remains to be seen; it could be one large slashing after sufficient strikes have accumulated, or it could be a slowly increasing penalty that gets quadratically more severe as more and more cheating behavior is detected.

With respect to monitoring a minipool's status, **we are working with the https://beaconcha.in maintainers** to include such information on the page for the minipool's corresponding validator.
As this is one of the most prominent eth2.0 block exploring sites and is frequently used for monitoring the status of a validator, we felt that this is one of the most promising options to notify node operators of a side-channel detection event.
This way, they have an opportunity to take corrective action.

As we discussed in the Analysis section, we expect the punishment of RPL and Beacon Chain balance slashing would serve as a strong deterrent against side channel and shadow mempool abuse.


## Option 2: Trusted Block Builders

Another potential strategy involves employing a variant of the [proposer/block builder separation design](https://ethresear.ch/t/proposer-block-builder-separation-friendly-fee-market-designs/9725) that Vitalik has suggested. We would use our MEV partners to **build entire blocks** for Rocket Pool node operators to propose.

In our variant, we have a three-party system: a trusted block builder, an untrusted proposer, and a trusted Oracle DAO.
Our block builder partners would publicly expose the hashes of the blocks they provided to the proposer, and the Oracle DAO would verify those blocks were proposed without any modifications.
This would mitigate false positives entirely.

We could evolve this functionality to another fraudproof system where a whistleblower did this validation independently, and was rewarded for doing so; this would be analogous to a Rocket Pool-specific implementation of the Beacon Chain's slashing system.
Offenders would be subject to the same strike system and countermeasure strategy defined above to mitigate bad behavior.

We are currently exploring an implementation of this solution with Flashbots.

The downside to this strategy is that Rocket Pool would **require** node operators to run an MEV-compatible execution client; they could no longer run a "normal" (non-MEV) execution client.
As we would only be looking at whether or not a node operator proposed a block that was built by a trusted partner, it would mean they would no longer be able to create their own blocks from the canonical Ethereum transaction pool (which is what a "normal" client would do).

That being said, an argument can be made that out-of-band problems require out-of-band solutions and this may be a case where fighting fire with fire is the best solution.
As MEV-augmented returns are expected to be considerably larger than basic Beacon Chain returns, it may be in the best interests of both the node operator *and the rETH staking pool* to require that an MEV-compatible execution client be used.


## Option 3: Wait and See

This is less of a proactive approach.
In this option, we simply wait to see what the Ethereum developers will do to combat MEV.
It's possible that the future of the protocol will have systems in place to mitigate MEV earnings so much that acting selfishly no longer holds an appreciable concern from the perspective of Rocket Pool's health.
The conversations so far have made this doubtful, but it is certainly an option we could take.

Rocket Pool is certainly going to need to adjust to Ethereum's development in the future; this option simply embraces that position to the fullest extent.
While we would rather be proactive than reactive, the above options may not be desirable by the Rocket Pool community due to the responsibilities they add to the Oracle DAO.


## Disclaimer

As an important note, we want to reiterate that **these are preliminary options that we are still exploring**.
We are not committing to these solutions at this time; we are simply offering them to the community as a catalyst to inspire further conversation about the problem and potential solutions. 
We look forward to the ensuing discussion about these options, their pros and cons, and other suggestions to mitigate the MEV-related challenges with decentralized node operation.
