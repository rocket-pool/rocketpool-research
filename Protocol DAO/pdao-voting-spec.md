# [WIP] On-Chain Protocol DAO System Specification

This specification describes the terminology, behavior, and design details of the Rocket Pool on-chain Protocol DAO proposal, voting, and challenge process.


## Rocket Pool Protocol Definitions

This specification touches on a wide variety of topics related to the Rocket Pool protocol and Merkle summation trees. This section formally describes concepts that are used in the context of the Rocket Pool protocol and its smart contract functionality.


### Block Number

Rocket Pool procotol DAO proposals are centered around the notion of a specific **block number**. Many proposal and voting functions require this as an argument.
The block number is the specific number of a block on the Execution layer that corresponds to the state used to create the proposal.
Any supplemental operations related to that proposal, such as challenging, must use the same corresponding block number when accessing the state of the chain for any reason (e.g., getting node information using `eth_call` routines). 


### Rocket Pool Nodes

A Rocket Pool **node** is an entity that has been formally registered with the Rocket Pool network (i.e., it has called the `rocketNodeManager.registerNode(...)` function).
Nodes are identified with two unique values:
1. An **index** (`uint256`)
2. An **address** (typical 20-byte Ethereum account address), which corresponds to the sender that called `registerNode()`.

Rocket Pool nodes are typically referred to by their address, but occasionally in this document, they will also be referred to by their index. To get the address of the node at index `i`, use the `rocketNodeManager.getNodeAt(i)` function.

Note that all Rocket Pool contract functions refer to a node by its address, so there is no corresponding view to get the index of a node by its address. However, if using `rocketNodeManager.getNodeAddresses(offset, limit)`, the address at index `j` in the resulting array corresponds to the node with index `i + offset`. 

Once registered with the network, a node can never be destroyed. It will always exist and both its index and address will remain constant.


### Delegation

The Rocket Pool protocol DAO allows nodes to "delegate" their voting power to another node, which allows that other node to vote on their behalf. Nodes **cannot** be "undelegated". Every node has a delegate. That being said, by default, a node's delegate is *its own address* which means it is solely responsible for assigning its own voting power during a vote.

A node's delegate is determined by the following function:

```go
delegateAddress := rocketNetworkVoting.getDelegate(nodeAddress, blockNumber)
```

where `nodeAddress` is the address of the node in question, `blockNumber` is the number of the Execution layer block whose state you want to query, and the resulting `delegateAddress` is the address of the node this node delegated its voting power to as of the provided block.


### Voting Power

Voting Power is a unitless quantity used to signify how much "power" a particular Rocket Pool node has relative to other nodes. When that node votes on a proposal, the node's voting power is assigned to the choice in the vote.

There are two flavors of voting power:

1. **Local voting power** - the raw amount of voting power the node itself has based on its effective RPL stake, This quantity ignores the node's voting delegation choice.
2. **Delegated voting power** - the aggregated amount of voting power across all of the nodes in the Rocket Pool network that have delegated their voting power to this node.

To get a node's local voting power, use the following function:

```go
localVP := rocketNetworkVoting.getVotingPower(nodeAddress, blockNumber)
```

where `nodeAddress` is the address of the node in question, `blockNumber` is the number of the Execution layer block whose state you want to query, and the resulting `localVP` is the total amount of local voting power this node had as of the provided block.

*Note this quantity will be provided in Wei (a fixed-point 18-decimal integer). It's important to keep it in integer-format during actual arithmetic to mitigate floating-point rounding errors during intermediate calculations. To translate it to a human-readable quantity for display purposes, divide it by 10^18 and treat the result as a floating-point number.*

A node's *delegated* voting power is not stored on-chain. It has to be derived off-chain by iterating over each node in the Rocket Pool network, keeping track of which nodes delegated to the node in question, calling `getVotingPower(...)` for each of those, and summing them all to arrive at the total.

Delegated voting power is calculated as a sum of the local voting power of the node's **direct delegates**; it does not support "chaining". More formally, in a situation where node `X` has delegated to node `Y`, and node `Y` has itself delegated to node `Z`. Node `Z`'s delegated voting power will include node `Y`'s, but it will *not* include node `X`'s. The voting power is not calculated by recursively traversing delegate assignments.


## Merkle Summation Tree Definitions

This section formally defines terms and concepts related to Merkle summation trees, as the Rocket Pool protocol DAO voting process uses them extensively as part of its on-chain verification process.


### Tree

A Merkle summation tree is a **binary tree** composed of individual tree nodes (which are further described below). In the context of a Merkle summation tree, a node can fall into one of three categories:

1. The **root** node, which is the single node at the top of the tree. With respect to Rocket Pool's usage, the root node will *always* have exactly two children.
2. **Intermediate** nodes, which are below the root but not at the bottom of the tree. With respect to Rocket Pool's usage, intermediate nodes *always* have exactly two children.
3. **Leaf** nodes, which are at the bottom of the tree. Each leaf node will *always* have exactly zero children.


### Node

A tree **node** is a single entity that exists within the tree. Tree nodes are *not* related to the previously defined Rocket Pool node concept, despite the common naming. The contents of a node (and how they are calculated) is defined by the type of node.

For **root and intermediate nodes**:
1. A **left child**, pointing to the left child node.
2. A **right child**, pointing to the right child node.
3. A **sum**, which is an integer value. It is calculated as:
    ```go
    node.sum = leftChild.sum + rightChild.sum
    ```
4. A **hash**, which is a 32-byte `keccak256` hash value. It is calculated as:
    ```go
    leftSum := leftChild.sum as [32]byte
    rightSum := rightChild.sum as [32]byte
    node.hash = keccak256(leftChild.hash ++ leftSum ++ rightChild.hash ++ rightSum)
    ```
    where the `++` operator represents concatenation of byte arrays. As all four arguments will be arrays of 32 bytes, the total being put into the `keccak256` function will be an array of 128 bytes.

For **leaf nodes**:
1. A **sum**, which is an integer value.
   - NOTE: `sum`'s exact value is context-specific and will be defined in the context of each tree in the voting protocol below.
2. A **hash** which is a 32-byte `keccak256` hash value. It is calculated as:
    ```go
    sumBytes := node.sum as [32]byte
    node.hash = keccak256(sumBytes)
    ```


### Level

A **level** within the Merkle summation tree, refers to a row of nodes (that is, all nodes at some depth `d`; in other words, all nodes that have exactly `d` parents above them in the tree.)

Levels are **0-indexed**, so the first level is Level 0.


### Index

Each tree node is assigned an **index**, which is a positive integer value uniquely identifying that node. Indices are assigned in a breadth-first fashion, counting from left-to-right first, then top-to-bottom second.

**Tree indices are 1-indexed**, not 0-indexed. The index of the root node is 1.


## Rocket Pool Voting Trees

In the protocol DAO proposal verification system, Rocket Pool employs **two types** of Merkle summation trees: **node** voting trees, and **network** voting trees. Both of them are structurally identical; the only difference is in the definition of the `sum` property for their leaf nodes.

Both types of trees share the following properties:
- A leaf node's level-specific index in the tree corresponds to the index of a Rocket Pool node. That leaf node represents the Rocket Pool node with the same index. For example: the first leaf node (the left-most node in the tree at the bottom row) corresponds to Rocket Pool node 0. The fifth leaf node corresponds to Rocket Pool node 4, and so on.
- The trees are *complete*, in the sense that the leaf row contains a number of tree nodes that is a power of two. If there are not a power-of-two number of Rocket Pool nodes in the network, the tree is *padded* with empty nodes to make the total number a power of two.
  - For example, if there were 13 Rocket Pool nodes, the next highest power of two is 16 so the first 13 leaf nodes in the tree would correspond to Rocket Pool nodes 0-12, and ttree would have three additional "padding" leaf nodes with "empty" data (defined later) which do not correspond to Rocket Pool nodes but are just there to complete the tree.
  - Thus, for both types of trees, the total number of nodes is *always* a power of two, minus one.


### Node Voting Tree

A **node voting tree** is a Merkle summation tree representing the *delegated* voting power of a single Rocket Pool node, called the `targetNode`. For a given `blockNumber`, there will be one of these trees for *each* Rocket Pool node in the network.

The `sum` of a leaf node is calculated as follows:
1. Get the 0-based, level-specific `leafIndex` of the leaf node (for example, `0` being the `leafIndex` of the left-most leaf node).
2. Get the address `nodeAddress` of the Rocket Pool node with this index:
   ```go
   nodeAddress := rocketNodeManager.getNodeAt(leafIndex)
   ```
3. Get the address `delegateAddress` of the Rocket Pool node's delegate:
   ```go
   delegateAddress := rocketNetworkVoting.getDelegate(nodeAddress, blockNumber)
   ```
4. If `delegateAddress` matches the address of `targetNode`:
   1. Get the node's local voting power `localVP`:
      ```go
      localVP := rocketNetworkVoting.getVotingPower(nodeAddress, blockNumber)
      ```
   2. Construct a new leaf node at index `leafIndex` with `localVP` as the value of `node.sum`.
5. If `delegateAddress` does not match the address of `targetNode`:
   1. Construct a new leaf node at index `leafIndex` with `0` value as the sum; that is, `node.sum = 0`.


### Network Voting Tree

A **network voting tree** is a Merkle summation tree representing the *delegated* voting power of the entire Rocket Pool network. For a given `blockNumber`, there will be exactly one of these trees total.

The `sum` of a leaf node is calculated as follows:
1. Get the 0-based, level-specific `leafIndex` of the leaf node (for example, `0` being the `leafIndex` of the left-most leaf node).
2. Get the address `nodeAddress` of the Rocket Pool node with this index:
   ```go
   nodeAddress := rocketNodeManager.getNodeAt(leafIndex)
   ```
3. Determine the Rocket Pool node's *delegated* voting power `delegatedVP` (for example, by creating a node voting tree using the above rules with `nodeAddress` as the `targetNode` and providing the final `sum` value of the root node).
4. Construct a new leaf node at index `leafIndex` with `delegatedVP` as the value of `node.sum`.


## Exemplary Setup

When walking through the process of proposing and verifying a proposal, it will greatly help to have a consistent Rocket Pool network setup we can use to explore various situations. The scenario here defines **13** nodes that make up an entire exemplary Rocket Pool network, along with their total voting power and delegate assignment choices:

| Node ID | Local Voting Power | Delegate |
| ------- | ------------------ | -------- |
| 0       | 385                | 5        |
| 1       | 294                | 1        |
| 2       | 849                | 9        |
| 3       | 618                | 9        |
| 4       | 294                | 4        |
| 5       | 0                  | 11       |
| 6       | 591                | 9        |
| 7       | 430                | 9        |
| 8       | 544                | 4        |
| 9       | 10                 | 9        |
| 10      | 544                | 5        |
| 11      | 903                | 11       |
| 12      | 108                | 5        |

The following notes are useful to observe when exploring this example:

- Nodes 1, 4, 9, and 11 have all delegated to themselves so they are responsible for voting on behalf of their constituents. The other nodes have each delegated to one of these.
- Node 8 has some local voting power, and it has delegated to node 4 which also has local voting power. To calculate their powers:
  - Node 8's local power is `544` and its delegated power is `0` (as no one has delegated to it).
  - Node 4's local power is `294` and its delegated power is `(294) + (544) = 838` where each node's voting power is represented by a set of parentheses, and the node itself's voting power is first in the list.
- Nodes 0, 10, and 12 are delegated to node 5. This makes node 5's delegated voting power `(0) + (385) + (544) + (108) = 1037`.
  - Node 5 itself is delegating to node 11. However, because delegated voting power does not follow a chain of delegates recursively, node 11's delegated voting power is only `(903) + (0) = 903`, **not** `(903) + (1037) = 1940`.
  - In other words, the voting power delegated to node 5 **does not** carry over to node 11; only the **local** voting power of node 11's *direct delegates* is applied to its delegated voting power. 
- Node 9 has several constituents deligating to it (nodes 2, 3, 6, 7, and itself). Its delegated voting power will be `(10) + (849) + (618) + (591) + (430) = 2498`.


To demonstrate the concepts thus far visually, below is a diagram representing the **node voting tree** for Rocket Pool node 9:

![](./node-tree-1.png)

where:
- The white numbers to the left of the image indicate each vertical **level** in the tree. Note that the number of tree nodes in each level equals `2^level`. 
- The orange numbers to the right of each tree node indicate the **tree index** of the tree node. As this quantity is 1-indexed, the root node has index 1.
- The green numbers to the left of each leaf node indicate the **node index** of the corresponding Rocket Pool node that is represented by that tree node. In our example, there are 13 Rocket Pool nodes so the first 13 leaf nodes correspond to each one sequentially; the remaining 3 leaf nodes at the end of level 4 are empty and used as padding. They do not correspond to Rocket Pool nodes.