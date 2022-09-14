# Determining a Node Operator's Fee Recipient via On-Chain Data

The expected fee recipient for any Rocket Pool validator (a "minipool") can be completely derived via on-chain data.
This is the specification for determining what a node operator's (and by extension, their minipools') fee recipient should be based on the current state of the Beacon and Execution chains.
Block builder relays can use this to detect if a validator corresponds to a Rocket Pool minipool and derive its proper fee recipient address instead of using whatever fee recipient is provided during registration.

This has two primary benefits:
- It prevents penalties on the node operator due to accidental misconfiguration
- It prevents intentional misconfiguration and theft of rewards ("cheating")

Essentially, the process for fee recipient derivation is broken into the following steps:
- Determine if the validator corresponds to a Rocket Pool minipool
- If so, retrieve the node address that owns the minipool
- Check if the node is opted into the Smoothing Pool
  - If so, use the Smoothing Pool contract address
  - If not, check if the node left the Smoothing Pool less than 2 epochs ago
    - If so, use the Smoothing Pool contract address (to prevent front-running exploits)
    - If not, retrieve and use the node's fee distributor contract address

Each of these steps is described in more detail below.


## Prerequisites

To perform this check, you will need:
- A synced Execution Client with RPC access
- A synced Consensus Client (Beacon Node) with standard Beacon API access


## Acquiring the Relevant Contract Addresses and ABIs

### RocketStorage

This process begins by retrieving the `RocketStorage` contract for the relevant network.
`RocketStorage` is effectively the "base" contract which contains addresses and variables for the rest of the protocol; everything can be retrieved from it.

The `RocketStorage` addresses for various networks are listed below:
- Mainnet: `0x1d8f8f00cfa6758d7bE78336684788Fb0ee0Fa46` ([Etherscan source + ABI](https://etherscan.io/address/0x1d8f8f00cfa6758d7bE78336684788Fb0ee0Fa46#code))
- Goerli-Prater: `0xd8Cd47263414aFEca62d6e2a3917d6600abDceB3` ([Etherscan source + ABI](https://goerli.etherscan.io/address/0xd8Cd47263414aFEca62d6e2a3917d6600abDceB3#code))


### Network Contract Addresses

You can acquire the **address** of a network contract as follows:
```
address := rocketStorage.getAddress(Keccak256("contract.address" + contractName))
```

For example, the `rocketMinipoolManager` contract address would be retrieved via:
```
rocketMinipoolManagerAddress := rocketStorage.getAddress(Keccak256("contract.addressrocketMinipoolManager"))
```


### Network Contract ABIs

You can acquire the **ABI** of a network contract as follows:
```
abiString := rocketStorage.getString(Keccak256("contract.abi" + contractName))
```

For example, the `rocketMinipoolManager` contract ABI would be retrieved via:
```
rocketMinipoolManagerAbiString := rocketStorage.getString(Keccak256("contract.abirocketMinipoolManager"))
```

The resulting string is **zlib compressed** and **encoded via base64**.
You must decode it in order to retreive the original JSON describing the contract's ABI.



## Required Contracts

You will need to acquire the following contracts (both addresses and ABIs unless otherwise specified):

- `rocketMinipoolManager`
- `rocketMinipool` (ABI only)
- `rocketNodeManager`
- `rocketSmoothingPool`
- `rocketNodeDistributorFactory`


## Determining if a Validator is a Rocket Pool Minipool

To check if a validator is part of the Rocket Pool network, use the `rocketMinipoolManager.getMinipoolByPubkey(pubkey)` function by providing the validator's pubkey as the only argument.

For example, to check validator [`0x9904d13c62f33b9b2c17ed0991e56c8e0337f632bbac1b789d282c228b20b93016d3a43d0c86788e8982c7d8c96b6a3f`](https://beaconcha.in/validator/0x9904d13c62f33b9b2c17ed0991e56c8e0337f632bbac1b789d282c228b20b93016d3a43d0c86788e8982c7d8c96b6a3f):

```
minipoolAddress := rocketMinipoolManager.getMinipoolByPubkey(0x9904d13c62f33b9b2c17ed0991e56c8e0337f632bbac1b789d282c228b20b93016d3a43d0c86788e8982c7d8c96b6a3f)
```

*Note that this function expects a byte array as the argument, not a string.*

If this returns `0x0000000000000000000000000000000000000000`, then the validator is **not** a Rocket Pool minipool.

If it returns any other valid address, it **is** a Rocket Pool minipool.
The address provided is the contract address of the minipool that corresponds to the provided validator.


## Retrieving the Node Address of a Minipool's Owner

Acquire the ABI of the `rocketMinipool` contract, and construct a new contract binding at the `minipoolAddress` retrieved in the previous step.

Call this function to retrieve the address of the owning node:

```
nodeAddress := rocketMinipool.getNodeAddress()
```


## Checking if a Node Operator is in the Smoothing Pool

Call the following function to check a node's Smoothing Pool status:

```
isOptedIn := rocketNodeManager.getSmoothingPoolRegistrationState(nodeAddress)
```

This will be `true` if they are opted in, or `false` if they are not.

If this returns `true`, then **the fee recipient should be the address of the `rocketSmoothingPool` contract**.


## Checking the Opt-Out Time to Prevent Front-Running

If the above returns `false`, you must verify that the user **didn't opt-out immediately upon seeing an upcoming proposal in the current or next epoch** - this is considered to be a front-running attack against the protocol.
Rocket Pool's Smartnode software is designed to prevent users from doing this unless they intentionally exploit it, in which case your relay can enforce the correct fee recipient and circumvent the attack.

This is done by checking the **opt-out time**, deriving the **Epoch** on the Beacon chain at the time of the opt-out, and retaining the Smoothing Pool contract address as the fee recipient until **the Epoch after the opt-out Epoch has passed**.

Check the opt-out time as follows:

```
optOutTime := rocketNodeManager.getSmoothingPoolRegistrationChanged(nodeAddress)
```

This value will be a Unix timestamp, in seconds.

If the value is `0`, the node never opted into the Smoothing Pool.
**The fee recipient should be the address of the node's fee distributor contract** (see below for details).

If the value is not `0`, the node opted into, and then opted out of, the Smoothing Pool.
It must be checked for this attack.

Derive the Epoch number at the opt-out time by using the standard Beacon API for [the Genesis information](https://ethereum.github.io/beacon-APIs/#/Beacon/getGenesis) and [the Beacon config](https://ethereum.github.io/beacon-APIs/#/Config/getSpec):

```
secondsSinceGenesis := optOutTime - genesis.data.genesis_time
optOutEpoch := secondsSinceGenesis / (beaconConfig.data.SECONDS_PER_SLOT * beaconConfig.data.SLOTS_PER_EPOCH)
```

Get the current Epoch from [the standard Beacon API](https://ethereum.github.io/beacon-APIs/#/Beacon/getBlockHeader) for the `head` state:

```
currentEpoch := header.data.header.message.slot / beaconConfig.data.SLOTS_PER_EPOCH
```

If `currentEpoch` is greater than `optOutEpoch + 1`, then **the fee recipient should be the address of the node's fee distributor contract** (see below for details).

If `currentEpoch` is less than or equal to `optOutEpoch + 1`, then **the fee recipient should be the address of the `rocketSmoothingPool` contract**.


## Getting the Node's Fee Distributor Contract Address

If the node has never opted into the Smoothing Pool, or has opted out and has passed the front-running cooldown window, then their fee recipient should be the node's unique fee distributor contract.

The fee distributor contract address can be retrieved as follows:

```
feeDistributorAddress := rocketNodeDistributorFactory.getProxyAddress(nodeAddress)
```
