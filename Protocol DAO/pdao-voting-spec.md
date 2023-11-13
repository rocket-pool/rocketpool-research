# [WIP] On-Chain Protocol DAO System Specification

This specification describes the terminology, behavior, and design details of the Rocket Pool on-chain Protocol DAO proposal, voting, and challenge process.


## Rocket Pool Protocol Definitions

This section formally describes concepts that are used in the context of the Rocket Pool protocol and its smart contract functionality.


### Block Number

Rocket Pool protocol DAO proposals are centered around the notion of a specific **block number**. Many proposal and voting functions require this as an argument.
The block number is the specific number of a block on the Execution layer that corresponds to the state used to create the proposal.
Any supplemental operations related to that proposal, such as challenging, must use the same corresponding block number when accessing the state of the chain or transacting on the proposal for any reason (e.g., getting node information using `eth_call` routines). Note that this is *not* the block number provided as an option in the `eth_call` parameters, but is rather an argument specifically passed to certain Rocket Pool contract functions that require it.


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
2. **Delegated voting power** - the aggregated amount of local voting power across all of the nodes in the Rocket Pool network that have delegated their voting power to this node.

To get a node's local voting power, use the following function:

```go
localVP := rocketNetworkVoting.getVotingPower(nodeAddress, blockNumber)
```

where `nodeAddress` is the address of the node in question, `blockNumber` is the number of the Execution layer block whose state you want to query, and the resulting `localVP` is the total amount of local voting power this node had as of the provided block.

*Note this quantity will be provided in Wei (a fixed-point 18-decimal integer). It's important to keep it in integer-format during actual arithmetic to mitigate floating-point rounding errors during intermediate calculations. To translate it to a human-readable quantity for display purposes, divide it by 10^18 and treat the result as a floating-point number.*

A node's *delegated* voting power is not stored on-chain. It has to be derived off-chain by iterating over each node in the Rocket Pool network, keeping track of which nodes delegated to the node in question, calling `getVotingPower(...)` for each of those, and summing them all to arrive at the total.

Delegated voting power is calculated as a sum of the local voting power of the node's **direct delegates**; it does not support "chaining". More formally, in a situation where node `X` has delegated to node `Y`, and node `Y` has itself delegated to node `Z`: node `Z`'s delegated voting power will include node `Y`'s, but it will *not* include node `X`'s. The voting power is not calculated by recursively traversing delegate assignments.


## Merkle Summation Tree Definitions

This section formally defines terms and concepts related to Merkle summation trees, as the Rocket Pool protocol DAO voting process uses them extensively as part of its on-chain verification process.


### Tree

A Merkle summation tree is a **binary tree** composed of individual tree nodes (which are further described below) and parent-child relationships between those nodes.


### Node

A tree **node** is a single entity that exists within the tree. Tree nodes are *not* related to the previously defined Rocket Pool node concept, despite the common naming. In the context of a Merkle summation tree, a node can fall into one of three categories:

1. The **root** node, which is the single node at the top of the tree. With respect to Rocket Pool's usage, the root node will *always* have exactly two children.
2. **Intermediate** nodes, which are below the root but not at the bottom of the tree. With respect to Rocket Pool's usage, intermediate nodes *always* have exactly two children.
3. **Leaf** nodes, which are at the bottom of the tree. Each leaf node will *always* have exactly zero children.
 
The contents of a node (and how they are calculated) is defined by the type of node.

For **root and intermediate nodes** the contents are:
1. A **left child**, pointing to the left child node **(not serialized)**.
2. A **right child**, pointing to the right child node **(not serialized)**.
3. A **sum**, which is an unsigned 256-bit (32-byte) big-endian integer value. It is calculated as:
    ```go
    node.sum = leftChild.sum + rightChild.sum
    ```
4. A **hash**, which is a 32-byte `keccak256` hash value. It is calculated as:
    ```go
    node.hash = keccak256(leftChild.hash ++ leftChild.sum ++ rightChild.hash ++ rightChild.sum)
    ```
    where the `++` operator represents concatenation of byte arrays. As all four arguments will be arrays of 32 bytes, the total being put into the `keccak256` function will be an array of 128 bytes.

For **leaf nodes** the contents are:
1. A **sum**, which is an unsigned 256-bit (32-byte) big-endian integer value.
   - NOTE: `sum`'s exact value is context-specific and will be defined in the context of each tree in the voting protocol below.
2. A **hash** which is a 32-byte `keccak256` hash value. It is calculated as:
    ```go
    node.hash = keccak256(node.sum)
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
  - For example, if there were 13 Rocket Pool nodes, the next highest power of two is 16 so the first 13 leaf nodes in the tree would correspond to Rocket Pool nodes 0-12, and the tree would have three additional "padding" leaf nodes with "empty" data (defined later) which do not correspond to Rocket Pool nodes but are just there to complete the tree.
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
   2. Construct a new leaf node at index `leafIndex` with `node.sum = localVP`.
5. If `delegateAddress` does not match the address of `targetNode`:
   1. Construct a new leaf node at index `leafIndex` with `node.sum = 0`.


### Network Voting Tree

A **network voting tree** is a Merkle summation tree representing the *delegated* voting power of the entire Rocket Pool network. For a given `blockNumber`, there will be exactly one of these trees total.

The `sum` of a leaf node is calculated as follows:
1. Get the 0-based, level-specific `leafIndex` of the leaf node (for example, `0` being the `leafIndex` of the left-most leaf node).
2. Get the address `nodeAddress` of the Rocket Pool node with this index:
   ```go
   nodeAddress := rocketNodeManager.getNodeAt(leafIndex)
   ```
3. Determine the Rocket Pool node's *delegated* voting power `delegatedVP` (for example, by creating a node voting tree using the above rules with `nodeAddress` as the `targetNode` and providing the final `sum` value of the root node).
4. Construct a new leaf node at index `leafIndex` with `node.sum = delegatedVP`.


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

<center>

![](./node-tree-1.png)

*Figure 1: Exemplary node voting tree for Rocket Pool node 9.*
</center>

where:
- The white numbers to the left of the image indicate each vertical **level** in the tree. Note that the number of tree nodes in each level equals `2^level`. 
- The orange numbers to the right of each tree node indicate the **tree index** of the tree node. As this quantity is 1-indexed, the root node has index 1.
- The green numbers to the left of each leaf node indicate the **node index** of the corresponding Rocket Pool node that is represented by that tree node. In our example, there are 13 Rocket Pool nodes so the first 13 leaf nodes correspond to each one sequentially; the remaining 3 leaf nodes at the end of level 4 are empty and used as padding. They do not correspond to Rocket Pool nodes.

Here is a diagram representing the **network voting tree** for this example setup:

<center>

![](./network-tree-1.png)

*Figure 2: Exemplary network voting tree.*
</center>

Similar to the node voting tree, each leaf node here corresponds to the same Rocket Pool node.

Observe that the root node for Rocket Pool node 9's voting tree shown above had a sum of `2498`, which is the same as the sum of tree node 25 in the network voting tree here (where tree node 25 is the leaf node that corresponds to Rocket Pool node 9).  

Also note that in some cases (such as with Rocket Pool node 2 / tree node 18) the leaf node has a sum of 0 even though the node has some local voting power; this is because the node does not have any *delegated* voting power (i.e., none of the Rocket Pool nodes in the network have delegated to it), and delegated voting power is what determines the value of the leaf node sums in the network voting tree.


## Virtual Indices

Occasionally, the Rocket Pool protocol will rely upon a tree or a subtree (see the Pollards section below) in which the root node's tree index has been reassigned; rather than starting at index 1, the root node will start at something else. The rest of the tree behaves normally with respect to the rules above; only the root node's index will change. This index is called a **virtual index**. Below is the graph for Rocket Pool node 9's node voting tree, but this time showing the *virtual* indices that will be used during actual operations:

<center>

![](./virt-indices-1.png)

*Figure 3: Node voting tree for Rocket Pool node 9, showing virtual indices instead of logical ones.*
</center>


Here each tree index, written in orange, has been replaced by its corresponding *virtual* index. The root node starts at index 25 instead of index 1, as 25 is the index corresponding to that node in the network voting tree. If we were looking at the node voting tree for Rocket Pool node 3, the root node would have virtual index 19.

*While in theory this can all be achieved by making a single monolithic tree with the "top half" being the network voting tree and the "bottom half" being the node voting tree for each node, in practice the size of such a tree is infeasible to use on modern systems. It's much more efficient to break the composition into these two trees, as node voting trees will only need to be generated for a single Rocket Pool node to demonstrate an illegal state during a proposal challenge rather than constructing a full node voting power tree for every Rocket Pool node on the network.*

The level labels show both the "logical" level (starting at 0) and the "virtual" level (starting at whichever virtual level is assigned to the root node upon tree creation).

To *recursively* identify virtual indices of a tree node:
- The current node's virtual index is `n`.
- The left child's virtual index is `2n`.
- The right child's virtual index is `2n+1`.

To *iteratively* identify the virtual indices of a tree node:
- The root node's virtual index is `n`.
- The target node's logical level is `l`.
- The target node's position in the tree row (its offset in that row) is `o`, 0-indexed.
- The target node's index is `n * 2^l + o`.


## Pollards

A feature leveraged extensively by the protocol is the generation and distribution of **pollards**. A pollard is essentially the leaf row of a *subtree* within a graph. Subtrees are identified by two things:
- A root node
- A depth (levels to capture, 0-indexed)

For example, if we wanted to generate the pollard of the network voting tree with depth 2 and root node 3, the resulting subtree would be highlighted below:

<center>

![](./pollard-1.png)

*Figure 4: Exemplary subtree and pollard selection.*
</center>

Here the subtree is highlighted with the orange box, starting with tree node 3 as its root. The row of "leaf nodes" (the smallest in the subtree) is at level 3, and the subtree's **pollard** consists of four nodes: 12, 13, 14, and 15.

By knowing which node to start from and how far down the tree to traverse, you can generate and interpret arbitrary pollards in a tree.


## Proposal Creation Process

A Protocol DAO proposal is a limited-life-time entity stored on the Execution layer that proposes calling a function within the `rocketDAOProtocolProposals` contract to modify a setting stored in the contracts that commands some behavior of the Rocket Pool protocol. Typically this is used to modify `bool` or `uint` settings, though in special situations other proposals can be created (such as inviting someone to or removing someone from the security council).

Proposal creation requires four things:
1. A human-readable `message` describing the proposal's behavior or its intent.
2. A `payload`, which is a complete ABI-packed byte array representing a payload to a transaction on the `rocketDAOProtocolProposals` contract (for example, an ABI-packed transaction payload for `proposalSettingBool`).
3. A `blockNumber`, indicating which block on the Execution layer will act as the target block for information about the network's voting power, as well as any subsequent challenges to the network voting power tree submitted with the proposal.
4. A pollard of tree nodes called `treeNodes`, which is a pollard of the network voting tree for block `blockNumber` using the tree's root node and a pre-defined depth.

The combined arguments above are passed to `rocketDAOProtocolProposals.propose(message, payload, blockNumber, treeNodes)`** to formally submit the proposal to the network.

***NOTE: This function no longer exists, update with the correct one once you know what it is.*


### Network Tree and Pollard Creation

Prior to submitting the proposal, the network voting tree for `blockNumber` must be generated. Doing so can be done using the same rules defined in the [Network Voting Tree](#network-voting-tree) section. When the tree is created, the proposer must extract a pollard from it.

The depth used for this pollard is a constant within the Rocket Pool contracts, known as `depthPerRound`. The expected value to use on Mainnet is **depth 5**. However, for our simplified example here, we will use a standard depth-per-round of **depth 2** for the sake of demonstration.

Using the network voting tree shown in Figure 2, with the root node as the target and a depth of 2, we will produce a proposal pollard consisting of tree nodes **4, 5, 6, and 7**:

<center>

![](./proposal-pollard.png)

*Figure 5: Pollard used for exemplary proposal consisting of tree nodes 4-7.*
</center>

Thus, the `treeNodes` argument in the proposal will be a four-element array of tree node structs corresponding to tree nodes 4 through 7. Since the left and right child properties of these nodes are not serialized, the only serialized properties to provide in the transaction payload are `sum` and `hash`, where `sum` is a 256-bit unsigned integer. Explicitly, the `treeNodes` argument will consist of the following bytes:

```go
0x0000000000000000000000000000000000000000000000000000000000000126 // Sum of node 4 (294)
0x58e061b04c72870751334701f316b368014cb68cc400a3b95dcdc3f3ec6c14c0 // Hash of node 4
0x0000000000000000000000000000000000000000000000000000000000000753 // Sum of node 5 (1875)
0x20d9b2ddcb52369953a69092d66ec4e8806b13b75bc215dab18b95b217c2c4fe // Hash of node 5
0x0000000000000000000000000000000000000000000000000000000000000d49 // Sum of node 6 (3401)
0x46a2c66663caf1e80cf718e32c803d2051bb65030e1f8c41dda2825076818329 // Hash of node 6
0x0000000000000000000000000000000000000000000000000000000000000000 // Sum of node 7 (0)
0x562870d479b7accb3cbf66f8f01213a0084eaf1d5924421356c64f59982e25e4 // Hash of node 7
```

Upon submission the Rocket Pool smart contracts will use this pollard to calculate the Merkle root of the network voting power tree (which is the same as the hash of the root node in this tree) and store it on-chain, as well as emit an event. This will be used should any challenges to the proposal arise. Note that while the Merkle root is stored on-chain, the Pollard itself is not; this must be retrieved using the event emitted with proposal submission. The event signature is as follows:

```go
TBD
```


### Challenge Process