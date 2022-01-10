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
  + https://github.com/bloxapp/eth2-staking-pools-research/blob/stage/dkg.md
+ Medium of communication? gossip?

### Deposit / Duties

+ Consensus / Aggregation / Leader Selection - Istanbul BFT?
+ Medium of communication - how do we do it while preserving privacy? gossip?
+ Latency tolerance of IBFT?
+ Minimum number of shares required and appropriate threshold

### Exiting

To avoid node operators taking others hostage:

+ Set staking periods (1yr, 2yr, 3yr, etc)
+ Presigned exits - who holds the presigned exit messages? how do they renew/expire?

### Distribution of Funds (after withdrawal)

+ This should work the same


## Cryptoeconomics

+ Validators without an ETH bond have little incentive to exit - liquidity effects?
+ Are validators punished for bad behaviour? What behaviour? How is it detected? What punishment?
+ How are validators incentivised? ETH commission? RPL? 
+ How do we ensure that no more than 1/3 are malicious? Thus reducing Ethereum security as it scales


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
