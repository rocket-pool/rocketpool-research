# Rewards Tree v8

[RPIP-30](https://rpips.rocketpool.net/RPIPs/RPIP-30) is ratified.
The RPIP directs the oDAO to adopt a rewards tree strategy that shifts away from linear RPL rewards between the eligiblity bounds.
Instead, a piecewise function will calculate a rewards 'weight' above a minimum bound.
The maximum bound will be removed.
One piece of the piecewise function calls for use of `ln()`, or an approximation of `ln()`, on inputs greater than or equal to 2.

## Implementation options

### 1. oDAO standardizes on IEEE-754 floats

This option is perhaps the most naive.

1) No on-chain support for IEEE-754 (Future SCs may want to validate the computation of reward weight for a fraud proof).
2) IEEE-754 is weak about ln() support, saying that `ln(x)` is a "recommended" mathematical operation which "must" round correctly.
3) Different languages and different architectures may not trivially arrive at the same results for a given input.

I recommend against any approach involving floats.

### 2. oDAO agrees on a lookup table over a range, e.g. \[10, 400\], and with a particular granularity, e.g. 3 sigfigs.

This option is straight forward.
A table can be generated and converted into a simple array in any language, and attached to the specification.
The main downside is that it needs to be bounded, so above a certain level of collateral, rewards will not increase.
This is probably acceptable, as we can set the boundary high enough to capture any rational amount of collateral, and can set the granularity low enough to keep the table compact.

The table would simply map collateral percentages of borrowed eth to node weights, and the mapped value would represent the result of the piecewise function defined in RPIP-30.

I don't favor this approach, as I believe it is nearly as complex as the next option, but with more downsides (lost granularity, and constant rewards above the upper bound of the table). We can mitigate the granularity issues with non-even spacing over the domain of the LUT, but that adds additional complexity (beyond, in my opinion, the complexity of the next option).

### 3. EVM-friendly approximation

This option is the one I am recommending, and involves using an [iterative approximation](https://en.wikipedia.org/wiki/Binary_logarithm#Iterative_approximation) of log2 to calculate `ln(x)` as `log2(x)/log2(e)`.

Existing solidity library [prb-math](https://github.com/PaulRBerg/prb-math) has a good reference implementation of `ln()` via iterative approximation that operates on unsigned 60x18 (60 digits with 18 decimal places), and it has a maximum error of `2**-60`, which is well beyond sufficiently small (less than 1 wei).
I'll be basing my suggested change to the spec on this approach.

## Suggested changes to the Rewards Tree spec

To apply RPIP-30, the [spec](https://github.com/rocket-pool/rocketpool-research/blob/master/Merkle%20Rewards%20System/rewards-calculation-spec.md) should be updated to add a transformation to the [Collateral Rewards](https://github.com/rocket-pool/rocketpool-research/blob/master/Merkle%20Rewards%20System/rewards-calculation-spec.md#collateral-rewards) section, and to eventually remove the `maxCollateral` calculations.

### Modifying the calculation

We jump ahead in the spec to the scaling of `nodeEffectiveStake`, immediately before this block:
```golang
nodeAge := targetElBlock.Timestamp - registrationTime
if (nodeAge < intervalTime) {
    nodeEffectiveStake = nodeEffectiveStake * nodeAge / intervalTime
}
```

We will call a new function for each node with the signature `int getNodeWeight(eligibleBorrowedEth, nodeStake, ratio, belowMin=nodeEffectiveStake==0)` to calculate a `nodeWeight`. 

Update the subsequent section to include scaling `nodeWeight` by registration time:
```patch
nodeAge := targetElBlock.Timestamp - registrationTime
if (nodeAge < intervalTime) {
    nodeEffectiveStake = nodeEffectiveStake * nodeAge / intervalTime
+   nodeWeight = nodeWeight * nodeAge / intervalTime
}
```

Finally, a new term, `totalNodeWeight`, the sum of all nodes' `nodeWeight` must be calculated. 

### `getNodeWeight` specification

First, check if `belowMin` is `true`. Return 0 if so. This will remove nodes below `minCollateral`, as the spec has set `nodeEffectiveStake` to 0 for them, as well as nodes with no RPL staked yet, if they have not already been filtered.

Calculate `stakedRplValueInEth = nodeStake * ratio / 1 Eth` (since `ratio` and `nodeStake` are both measured in wei, they must be divided by 1 Eth to normalize the result).

Calculate `percentOfBorrowedEth = stakedRplValueInEth * 100 Eth / eligibleBorrowedEth` and if `percentOfBorrowedEth <= 15 Eth` return `100 wei * stakedRplValueInEth`. This handles the piecewise section of the function between 10 and 15 percent of borrowed.

Finally, if we have not yet returned, return `((13.6137 Eth + 2 wei * ln(percentOfBorrowedEth - 13 Eth)) * eligibleBorrowedEth) / 1 Eth`. The natural log `ln` is defined in the next section, and returns a value in wei with 18 decimal places. The final division is, again, to normalize, as the result of the outermost multiplication produces a result in Eth-squared.

### `ln` specification

The `ln(x)` function is specified to equal `log2(x) * 1 Eth / 1442695040888963407 wei`. The constant is the result of `log2(e)`. The `log2` function is defined in the next section, and returns a value in wei.

### `log2` specification

`log2(x)` is calculated using iterative approximation for 60 loops, ensuring an error of at most `2**-60`. 

Define `result = 0`

First, calculate `exponent` of the highest power of two that is less than or equal to the input `x`.
The easiest way to do this is to find the index of most significant set bit, with the least significant digit being 0-indexed.
Id est, if `x` is `40 Ether`, the highest power of two that is less than 40 Eth is 32 Eth, and the most significant bit of 0b0100000 (32 in binary) is at index 5, so `exponent` is 5.

Multiply the `exponent` by 1 Eth and add it to the `result`, `result = result + exponent * 1 Eth`.
Next, calculate the iterative approximation's `y` term `y = x >> exponent`. If `y` is 1 Eth, return `result`.

Define `delta = 1 Eth`.

Loop 60 times. In each loop:
1. Divide delta by 2 wei, `delta = delta / 2 wei`
1. Square y, `y = y * y /  1 Eth`
1. If `y >= 2 Eth`:
    * Add `delta` to `result`, i.e. `result = result + delta`
    * Divide `y` by 2 wei, i.e. `y = y / 2 wei` 

After 60 loops, return `result`

## Phase-In period

RPIP-30 is going to be phased in over 5 rewards cycles, reaching full effect during the sixth.

Define `C` to be an integer in the closed range \[1, 6\] wei that represents the rewards cycle (offset by the cycle before the spec is adopted). That is, if the spec is adopted for rewards cycle 17, C is 1 when calculating cycle 17, 2 when calculating cycle 18, and so on until cycle 22, when `C` reaches and remains at 6.

For each node, `nodeCollateralAmount` will be defined by `(collateralRewards * C * nodeWeight / (totalNodeWeight * 6 wei)) + (collateralRewards * (6 wei - C) * nodeEffectiveStake / (totalEffectiveRplStake * 6 wei))`

The sum of `nodeCollateralAmount` will be used as `totalCalculatedCollateralRewards` in the subsequent `epsilon` comparison.

Once `C` reaches 6, the phase-in is complete, and `nodeCollateralAmount` is defined by `collateralRewards * nodeWeight / totalNodeWeight`.

In the final state of RPIP-30, max collateral is no longer needed and can be removed.

`maxCollateral` is currently given by `maxCollateral := eligibleBondedEth * maxCollateralFraction / ratio`. Once RPIP-30 removes the maximum, this term no longer needs to be calculated. `eligibleBondedEth` and `maxCollateralFraction` can also be removed, as they are only referenced in calculating the `maxCollateral` term.

Additionally, `nodeEffectiveStake` can be removed, provided the minimum is still applied to `nodeStake` before calculating `nodeWeight`.

## Updates to the merkle tree

I'm recommending adding `totalNodeWeight` to the merkle tree.

It isn't required for the distribution of rewards.
However, we will inevitably have to update things like the [Node Operator APR calculator](https://rocketpool.net/node-operators) or build other tools that can estimate the APR for a given RPL / Borrowed Eth configuration for node operators. Id est, a Node Operator deciding how much RPL to stake will want to know what kind of rewards to expect for various configurations.

In the old system, the APR could be approximated with just the `collateralRewards`, `nodeEffectiveStake`, and `totalEffectiveRplStake` terms. `collateralRewards` is reasonably predictable, `nodeEffectiveStake` comes from the inputs to the calculator, and `totalEffectiveRplStake` can be looked up on-chain (and adjusted by the input of the calcuator).

However, in the new system, a node operator deciding how much RPL to stake will have a rewards amount impacted by `totalWeight` in addition to `nodeEffectiveStake` and `eligibleBorrowedEth`.
The simplest way to update these tools would be to calculate the hypothetical `nodeWeight` and use a recent interval tree's `totalWeight` field to approximate what their share would be (`collaterapRewards * nodeWeight / (nodeWeight + totalWeight)`).