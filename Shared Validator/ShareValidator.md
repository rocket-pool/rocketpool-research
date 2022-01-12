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
+ Medium of communication? gossip? - gossip can be used. SSV.Network should have a DKG integration the next few months that uses the existing networking and uses pubsub

### Deposit / Duties

+ Consensus / Aggregation / Leader Selection - Istanbul BFT? QBFT(latest version of IBFT) yes
+ Medium of communication - how do we do it while preserving privacy? gossip or pubsub
+ Latency tolerance of IBFT? - should be a few hundreds of milliseconds. In comparison, BFT protocols like Cosmos have a 100 validators and produce a block in a second or so
+ Minimum number of shares required and appropriate threshold 
  + for fault tollerance you'd need 3f+1 shares (so 4,7,10,etc)
  + for collusion resistance the number would need to be quite high
  + we may also need to consider how operators are assigned to validators - randomisation?

### Exiting

To avoid node operators taking others hostage:

+ Set staking periods (1yr, 2yr, 3yr, etc)
+ Presigned exits - who holds the presigned exit messages? how do they renew/expire? how could these be generated without broadcasting?
  + oDAO members could hold presigned exits distributed between multiple members that will need to reconstruct if all else fails.

### Distribution of Funds (after withdrawal)

+ This should work the same


## Cryptoeconomics

+ Validators without an ETH bond have little incentive to exit - liquidity effects?
+ Are validators punished for bad behaviour? What behaviour? How is it detected? What punishment?
+ How are validators incentivised? 
  + ETH commission? 
  + RPL? 
  + In SSV.Network's case in SSV, no rev share at all.
+ How do we ensure that no more than 1/3 are malicious? Thus reducing Ethereum security as it scales
  + An RPL bond would still have to be required for permissionless unbonded minipool but is that enough? 
  + The idea is to reduce the bond how low could we go? 
  + RPL bonds are not as efficient as ETH bonds because they require the auction mechanism. 
  + We could just allow ODAO members to perform unbonded minipools as before?
+ If you end up in a group of bad node operators, you will be punished even though you are performing well


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
+ Vitalik has said that if the attestation process could be optimised then it may be possible to reduce the ETH deposit to 4 ETH 
  + The primary reason for using SSV would be to reduce collateral requirements so it would no longer be necessary in this case
  + The secondary reason is resilience but time will tell whether this is an issue - it is currently not an issue
