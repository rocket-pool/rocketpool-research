# Integration with Beaconcha.in

This is the design document for Rocket Pool's integration with https://beaconcha.in. 
It describes the Rocket Pool-specific metrics and alerts that would be useful if integrated with the block explorer's existing display and notification functions.

As requested, all of the below descriptions leverage [Rocket Pool's Go bindings](https://github.com/rocket-pool/rocketpool-go/) for its smart contracts.


## Background

Rocket Pool's architecture revolves around the concept of a **minipool**.
Minipools are contracts that live on the Eth1 chain.

When a node operator creates a new Beacon Chain validator using Rocket Pool,
the Smartnode stack will deploy a new minipool contract and set the validator's withdrawal credentials to the minipool's contract address using [the `0x01` prefix](https://github.com/ethereum/eth2.0-specs/pull/2149).
That minipool and validator are linked together for the life of the validator.

The minipool is responsible for several things, including managing the distribution of funds from the corresponding Eth2 validator once the validator exits the Beacon Chain and its balance is returned to the minipool on Eth1.

Minipools are created and owned by **nodes**, which refer to Eth1 addresses that have registered with the Rocket Pool network.
The user running the node is the **node operator**.
A node can have multiple minipools - one per validator.


## Using the Go Library

To use the Go library, start by adding it as a dependency to your project's `go.mod`:

```go
require (
	github.com/rocket-pool/rocketpool-go v1.0.0-rc3.0.20210728053744-fb47c4c5e2dd
    ...
)
```

**Note that the version of `rocketpool-go` is temporary; it will be updated to `v1.0.0-rc4` with the launch of the Prater test network on August 2nd, and will be updated again to v1.0.0 with the release of Rocket Pool on mainnet.*

The following sample program will demonstrate a quick proof-of-concept that shows several key characteristics involved in working with the Go binding.
In it, we'll map a Beacon Chain validator by its public key to its minipool contract address on Eth1 and get some details about the minipool.

For our example, we'll use [validator 215550](https://prater.beaconcha.in/validator/0xa13d17279b6512190c6843a0bbfc012ba70586713a311fd10155d2b3b61011a7aaa1df66ab5808345fb4fcba46f43e2e#attestations) on the Prater testnet, which is owned by [a corresponding minipool](https://goerli.etherscan.io/address/0xD22210e7e6D77488B12A7532eb1588E8a16f3CB5) on Goerli.

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/rocket-pool/rocketpool-go/minipool"
	"github.com/rocket-pool/rocketpool-go/rocketpool"
	"github.com/rocket-pool/rocketpool-go/types"
)

func main() {
    // Set up the arguments
    eth1Address := "http://my-eth1-client:port"
    rocketStorage := "hex-string-address-of-rocket-storage-contract"

    // Connect to ETH1
    ethClient, err := ethclient.Dial(eth1Address)
    if err != nil {
        fmt.Printf("Error connecting to ETH1 client on %s: %s\n", eth1Address, err)
        os.Exit(1)
    }

    // Create RP manager
    rp, err := rocketpool.NewRocketPool(ethClient, common.HexToAddress(rocketStorage))
    if err != nil {
        fmt.Printf("Error getting RocketStorage contract at address %s: %s\n", rocketStorage, err)
        os.Exit(1)
    }

    // Convert a validator public key string into its Rocket Pool representation
    // Note the pubkey does not have the 0x prefix!
    pubkeyString := "a13d17279b6512190c6843a0bbfc012ba70586713a311fd10155d2b3b61011a7aaa1df66ab5808345fb4fcba46f43e2e"
    pubkey, err := types.HexToValidatorPubkey(pubkeyString)
    if err != nil {
        fmt.Printf("Error getting validator public key for [%s]: %s\n", pubkeyString, err)
        os.Exit(1)
    }

    // Get the ETH1 address for the minipool that owns the validator
    minipoolAddress, err := minipool.GetMinipoolByPubkey(rp, pubkey, nil)
    if err != nil {
        fmt.Printf("Error getting minipool address for validator [%s]: %s\n", pubkeyString, err)
        os.Exit(1)
    }
    emptyStruct := common.Address{}
    if minipoolAddress == emptyStruct {
        fmt.Printf("Validator %s is not a Rocket Pool validator.\n", pubkeyString)
        os.Exit(1)
    } else {
        fmt.Printf("Minipool address for validator %s is %s.\n", pubkeyString, minipoolAddress.Hex())
    }

    // Create a new binding for the minipool at that address
    minipool, err := minipool.NewMinipool(rp, minipoolAddress)
    if err != nil {
        fmt.Printf("Error getting binding for minipool %s: %s\n", minipoolAddress.Hex(), err)
        os.Exit(1)
    }

    // Get the address of the node that owns the minipool
    node, err := minipool.GetNodeAddress(nil)
    if err != nil {
        fmt.Printf("Error getting node address for minipool %s: %s\n", minipoolAddress.Hex(), err)
        os.Exit(1)
    }
    fmt.Printf("Owning node: %s\n", node.Hex())

    // Get the status details for the minipool
    statusDetails, err := minipool.GetStatusDetails(nil)
    if err != nil {
        fmt.Printf("Error getting status details for minipool %s: %s\n", minipoolAddress.Hex(), err)
        os.Exit(1)
    }

    // Print the details
    fmt.Printf("Status for %s as of Block %d, %s: %s\n\n", 
        minipoolAddress.Hex(), 
        statusDetails.StatusBlock, 
        statusDetails.StatusTime.Format(time.RFC822),
        types.MinipoolStatuses[statusDetails.Status])
}
```

This will be your primary interface for interacting with the network.


### Creating a Rocket Pool Manager

Your primary interface for the Rocket Pool network is going to be the `rocketpool.RocketPool` type.
It is created in this section using the `rocketpool.NewRocketPool()` function:

```go
// Set up the arguments
eth1Address := "http://my-eth1-client:port"
rocketStorage := "hex-string-address-of-rocket-storage-contract"

// Connect to ETH1
ethClient, err := ethclient.Dial(eth1Address)
if err != nil {
    fmt.Printf("Error connecting to ETH1 client on %s: %s\n", eth1Address, err)
    os.Exit(1)
}

// Create RP manager
rp, err := rocketpool.NewRocketPool(ethClient, common.HexToAddress(rocketStorage))
if err != nil {
    fmt.Printf("Error getting RocketStorage contract at address %s: %s\n", rocketStorage, err)
    os.Exit(1)
}
```

You will need to pass in an `ethclient.Client` object which is connected to an Eth1 client on the appropriate network (Goerli for the testnet, or mainnet once it's released).
This is created using the `ethclient.Dial()` function from `go-ethereum`, which we assume you are already familiar with.

The second parameter is created using the hex string of the `RocketStorage` contract.
You can retrieve this address from the [Smartnode configuration file](https://github.com/rocket-pool/smartnode-install/blob/master/amd64/rp-smartnode-install/network/prater/config.yml#L2) for the appropriate network (our Prater testnet is linked here).

**Note that the contract address is currently not published; it will be added when we officially launch our Prater testnet on August 2nd.*


### Getting the Minipool Address of a Beacon Validator

The next section of code shows how to retrieve the Eth1 minipool address that corresponds to an Eth2 validator's public key:

```go
// Convert a validator public key string into its Rocket Pool representation
// Note the pubkey does not have the 0x prefix!
pubkeyString := "a13d17279b6512190c6843a0bbfc012ba70586713a311fd10155d2b3b61011a7aaa1df66ab5808345fb4fcba46f43e2e"
pubkey, err := types.HexToValidatorPubkey(pubkeyString)
if err != nil {
    fmt.Printf("Error getting validator public key for [%s]: %s\n", pubkeyString, err)
    os.Exit(1)
}

// Get the ETH1 address for the minipool that owns the validator
minipoolAddress, err := minipool.GetMinipoolByPubkey(rp, pubkey, nil)
if err != nil {
    fmt.Printf("Error getting minipool address for validator [%s]: %s\n", pubkeyString, err)
    os.Exit(1)
}
emptyStruct := common.Address{}
if minipoolAddress == emptyStruct {
    fmt.Printf("Validator %s is not a Rocket Pool validator.\n", pubkeyString)
    os.Exit(1)
} else {
    fmt.Printf("Minipool address for validator %s is %s.\n", pubkeyString, minipoolAddress.Hex())
}
```

This uses the `types.HexToValidatorPubkey()` function to convert the public key from its string representation to a format that Rocket Pool can use.
It then uses the `minipool.GetMinipoolByPubkey()` function to retrieve the corresponding minipool address.

The output of this portion will be as follows:

```
Minipool address for validator a13d17279b6512190c6843a0bbfc012ba70586713a311fd10155d2b3b61011a7aaa1df66ab5808345fb4fcba46f43e2e is 0xD22210e7e6D77488B12A7532eb1588E8a16f3CB5.
```

If you provide a public key that is *not* owned by Rocket Pool, `minipool.GetMinipoolByPubkey()` will return the `0x0` address.
The above sample checks for that and provides feedback accordingly.


### Getting the Owning Node Address

Once you have the minipool contract address, you can interact with the minipool using the `minipool` package.
This section shows how to get the owning node address for the minipool.

```go
// Create a new binding for the minipool at that address
minipool, err := minipool.NewMinipool(rp, minipoolAddress)
if err != nil {
    fmt.Printf("Error getting binding for minipool %s: %s\n", minipoolAddress.Hex(), err)
    os.Exit(1)
}

// Get the address of the node that owns the minipool
node, err := minipool.GetNodeAddress(nil)
if err != nil {
    fmt.Printf("Error getting node address for minipool %s: %s\n", minipoolAddress.Hex(), err)
    os.Exit(1)
}
fmt.Printf("Owning node: %s\n", node.Hex())
```

This begins with a call to `minipool.NewMinipool()` which constructs a binding for the minipool.
Any time you want to interact directly with a Minipool contract, you'll want to use this binding.

The next line calls `minipool.GetNodeAddress()` to retrieve the address of the Rocket Pool node that created this minipool.


### Getting Details about the Minipool

Another thing you can do with the `minipool.Minipool` binding is query as this next section demonstrates:

```go
// Get the status details for the minipool
statusDetails, err := minipool.GetStatusDetails(nil)
if err != nil {
    fmt.Printf("Error getting status details for minipool %s: %s\n", minipoolAddress.Hex(), err)
    os.Exit(1)
}

// Print the details
fmt.Printf("Status for %s as of Block %d, %s: %s\n\n", 
    minipoolAddress.Hex(), 
    statusDetails.StatusBlock, 
    statusDetails.StatusTime.Format(time.RFC822),
    types.MinipoolStatuses[statusDetails.Status])
```

In general, interacting with the Minipool itself can be done with this `minipool.Minipool` binding.
You will likely use that and the `node` package to interact with the owning node to achieve the bulk of the proposed metrics and notifications below.


## Proposed Metrics

This section describes minipool metrics that would be useful to display on the block explorer.

*TBD*


## Proposed Notifications

This section describes notifications related to the minipool or its owning node that would be useful to display on the block explorer.


### New RPL Rewards are Available

This checks to see if new RPL rewards are available to the node operator after Rocket Pool crosses a rewards checkpoint (schedule for every 3 days on Prater, and every 28 days on mainnet).

By default, the Smartnode stack will automatically claim these rewards for the node operator.
It checks to see if they are available every 5 minutes, and if so, attempts to claim them.
However, there are a few situations where automatic claiming will not work:

1. The node operator explicitly disables it, either in the configuration files or by stopping the `rocketpool_node` Docker container
2. The node operator sets a gas price threshold for automatic operations in the configuration files, and the network's current recommended gas price is greater than this threshold
3. The node wallet does not have enough ETH left to pay for the transaction's gas cost

In this case, the user should be notified that rewards are available but have not been claimed, so they can take action if desired.

To query the status of RPL rewards, you can use the following code:

```go
import (
	"github.com/rocket-pool/rocketpool-go/rewards"
	"github.com/rocket-pool/rocketpool-go/utils/eth"
    ...
)
...

rewardsAmountWei, err := rewards.GetNodeClaimRewardsAmount(rp, nodeAddress, nil)
rewards := eth.WeiToEth(rewardsAmountWei)
```

The `rewards.GetNodeClaimRewardsAmount()` function takes in a `rocketpool.RocketPool` manager object and the address of the owning node.
It returns how much RPL is available for the node to claim, denominated in Wei (thus multiplied by `10^18`).
The helper function `eth.WeiToEth()` converts this into the true representation of how much RPL is actually available as a `float64`.

A similar function, `rewards.GetTrustedNodeClaimRewardsAmount()`, can be called to get the rewards available to Oracle DAO members as part of their service:

```go
import (
	"github.com/rocket-pool/rocketpool-go/rewards"
	"github.com/rocket-pool/rocketpool-go/utils/eth"
    ...
)
...

rewardsAmountWei, err := rewards.GetTrustedNodeClaimRewardsAmount(rp, nodeAddress, nil)
rewards := eth.WeiToEth(rewardsAmountWei)
```

The parameters and return value are identical to `rewards.GetNodeClaimRewardsAmount()`.

Node operators have until the next checkpoint to claim these rewards.
Our request is to monitor these values, notice when they've become larger than 0 (thus meaning new rewards are available), and notify the user that a new claim is available and indicate the amount of RPL.

When either value drops back down to 0, meaning the respective rewards have been claimed, the user should be notified of a successful claim as well.
We can further explore this notification to link to the transaction on Eth1 showing the total gas cost of the transaction and the total RPL claimed.