# Shared Validator


## Aim 

+ Increase node operator supply by lowering capital requirements
+ Increase resilience by using threshold signatures

## Security Guarantees

+ Node operators must not be able to steal pool funds
+ Node operators must not be able to hold each other hostage (I want to exit but others don't)
+ Node operators should still be able to join permissionlessly
+ Node operator privacy should be maintained - no direct communication by IP
+ Implementing a permissionless shared validator should not affect Ethereum's overall security

## Process

### Setup

+ What distributed key generation (DKG) does DVT use? Joint-Feldman protocol? (interactive? what process? confirm secure under public comms)
  + https://github.com/bloxapp/eth2-staking-pools-research/blob/stage/dkg.md - Pederson for dkg (https://link.springer.com/content/pdf/10.1007%2F3-540-48910-X_21.pdf). Re-sharing is also possible.
+ Medium of communication? gossip? - gossip can be used. SSV.Network should have a DKG integration the next few months that uses the existing networking whic uses pubsub

### Deposit / Duties

+ Consensus / Aggregation / Leader Selection - Istanbul BFT? QBFT(latest version of IBFT) yes
+ Medium of communication - how do we do it while preserving privacy? gossip? - pubsub
+ Latency tolerance of IBFT? - should be a few hundreds of milliseconds. In coparison, BFT protocols like Cosmos have a 100 validators and produce a block in a second or so
+ Minimum number of shares required and appropriate threshold - for fault tollerance you'd need 3f+1 shares (so 4,7,10,etc)

### Exiting

To avoid node operators taking others hostage:

+ Set staking periods (1yr, 2yr, 3yr, etc)
+ Presigned exits - who holds the presigned exit messages? how do they renew/expire? You could have the oDAO hold presigned exits distribuited between multiple members that will need to reconstruct if all else fails. How do you solve it now? with a single operator?

### Distribution of Funds (after withdrawal)

+ This should work the same


## Cryptoeconomics

+ Validators without an ETH bond have little incentive to exit - liquidity effects?
+ Are validators punished for bad behaviour? What behaviour? How is it detected? What punishment?
+ How are validators incentivised? ETH commission? RPL? In SSV.Network's case in SSV, no rev share at all.
+ How do we ensure that no more than 1/3 are malicious? Thus reducing Ethereum security as it scales - If you bond validators then you have economical guarantess. If you don't you'll need some kind of a "verified" set of operators


## Background

+ https://medium.com/coinmonks/eth2-secret-shared-validators-85824df8cbc0
+ https://www.youtube.com/watch?v=awBX1SrXOhk
+ https://github.com/bloxapp/ssv
+ https://twitter.com/StakeETH/status/1466536673626316802
+ https://github.com/ethereum/EIPs/issues/650
+ IBFT - https://arxiv.org/pdf/2002.03613.pdf
+ https://arxiv.org/pdf/1809.03421.pdf
+ https://notes.ethereum.org/@vbuterin/rkhCgQteN?type=view#Why-32-ETH-validator-sizes
+ https://github.com/bloxapp/eth2-staking-pools-research/tree/35769df9f2ca7e68d14a42007b33d0ac8759594c
+ https://www.youtube.com/watch?v=Jtz9b7yWbLo


## Considerations

+ PBS - https://ethresear.ch/t/proposer-block-builder-separation-friendly-fee-market-designs/9725
+ Vitalik has said that if the attestation process could be optimised then it may be possible to reduce the ETH deposit to 4 ETH - Would the main use case for SSV in this case will be to reduce collateral requirements? 
