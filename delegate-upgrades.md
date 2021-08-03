# Rocket Pool Minipool Delegate Upgrade System

Our Prater testnet came with a number of changes to make Rocket Pool safer for node operators. One of these changes was the introduction of optional delegate contract upgrades.
This article will go into details about how this system works and how node operators can use these new features to their advantage.

## What is a Minipool Delegate Contract?

Every minipool in the Rocket Pool network has a matching smart contract deployed to the Ethereum 1 chain and therefore it's own Ethereum address. When a node operator 
deposits 16 or 32 ETH via `rocketpool node deposit` CLI command, a call is made to a factory contract which in turn deploys a minipool contract. The purpose of this contract
is to act as the receiving address for future beacon chain withdrawals. The minipool address is given as the withdrawal credential for the beacon chain deposit.
In order to allow the functionality of these contracts to be upgraded at a future date (either to fix bugs or add features), we utilise an Ethereum mechanism known
as `delegatecall`. The smart contract which is deployed is a just a proxy contract that redirects any calls made to it onto another contract which contains the
actual logic. This other contract is what we refer to as a delegate contract. With this technique, we are able to upgrade the logic of every minipool at once simply
by deploying one replacement delegate contract and telling the proxy contracts to point at the new one instead.

## What's the Problem?

This method works great and has some very desirable traits. But one glaring problem is that whoever holds the authority to upgrade this contract has immense power over every
deposit on the network. Previously, the Rocket Pool oDAO possessed the power to upgrade the delegate contract of every minipool at will. This made the oDAO a massive target
for malicious actors and does not offer node operators much guarantee about the security of their deposits.

## How We've Addressed the Problem

We need a solution which still allows us to upgrade minipool functionality without giving unlimited power to the oDAO. This requires some cooperation between node operators
and the oDAO. The system we have designed achieves this by allowing the oDAO to vote on a new delegate contract as was previously done. But node operators are now under no
obligation to accept the new contract for their minipools. Node operators can check the new delegate contract to ensure there is no foul play and if they accept the changes,
they can choose to upgrade their minipools to this new version. In addition, node operators can rollback to their previous delegate contract in case they change their mind.
And finally, node operators can configure their minipools to always just use the latest delegate contract proposed by the oDAO if they are trusting and want to reduce their
on-going maintenance requirements.

## New CLI Commands

Our smartnode software has been updated to include CLI commands to access these new features as a node operator of the Rocket Pool network. These new commands are explained
below:

### rocketpool minipool status

The minipool status command has been updated to include "Delegate address", "Rollback delegate", "Effective delegate", and "Use latest delegate" settings in the minipool
status readouts.

| Setting | Description |
| ------- | ----------- |
| Delegate address | This is the current delegate address your minipool is storing. When a minipool is first created, this value is set to the latest oDAO proposed delegate contract. If you upgrade your delegate contract, this value will be updated to the most recently proposed delegate contract instead. |
| Rollback delegate | This setting will initially be set to <none>. After upgrading your delegate, this setting will be set to whatever your previous "Delegate address" was set to. Rolling back will make your minipool go back to using this contract. |
| Effective delegate | This is just showing you which delegate contract your minipool is currently using. If you have enabled "Use latest delegate" then this will refer to the latest oDAO proposed contract, otherwise it will refer to your "Delegate address". |
| Use latest delegate | This setting is initially set to "no". If enabled, your minipool will always use the latest oDAO proposed delegate. |
 
### rocketpool minipool delegate-upgrade

This command will allow you to upgrade your delegate address for one or more of your minipools. You will be prompted which minipool you want to upgrade or given the option to upgrade
them all at once. 
  
### rocketpool minipool delegate-rollback

This command will allow you to roll back your delegate address for one or more of your minipools. You will be prompted which minipool you want to rollback or given the option to roll
them all back at once.

Please node: We only keep track of one previous delegate contract. You can not rollback multiple times. Once you upgrade a minipool, you loose the ability to rollback to the delegate contract
prior to your previous one.
  
### rocketpool minipool set-use-latest-delegate [yes|no]

This command allows you to enable or disable the 'Use latest delegate' setting on one or more of your minipools. You will be prompted for which minipool you want to set or given the option
to set it for all. You must specify yes or no when running the command. If you enable this setting, your 'Delegate address' is ignored and your minipool will always use the latest oDAO proposed contract.

Please note: If enabled, your minipool will automatically and immediately use the latest oDAO proposed contract without any interaction from you. Your 'Delegate address' will be ignored.

## Technical Implementation
  
`RocketMinipool` is the proxy contract which delegatecalls the `RocketMinipoolDelegate` contract which contains the minipool logic. When a `RocketMinipool` is first deployed, it makes a call to
Rocket Pool to retrieve the address of the latest `RocketMinipoolDelegate` and stores it in it's own storage under `rocketMinipoolDelegate`. It also defaults `useLatestDelegate` from it's own
storage to false. 
  
The fallback method on `RocketMinipool` checks if `useLatestDelegate` is false, and if so performs a delegatecall into it's stored `rocketMinipoolDelegate`. If it's true,
it queries the latest contract from Rocket Pool and performs a delegatecall into that instead.

```js
    // Delegate all other calls to minipool delegate contract
    fallback(bytes calldata _input) external payable returns (bytes memory) {
        // If useLatestDelegate is set, use the latest delegate contract
        address delegateContract = useLatestDelegate ? getContractAddress("rocketMinipoolDelegate") : rocketMinipoolDelegate;
        (bool success, bytes memory data) = delegateContract.delegatecall(_input);
        if (!success) { revert(getRevertMessage(data)); }
        return data;
    }
```

`RocketMinipool` now features a few extra functions which are handled by itself as opposed to handing them off to the delegate. These include `delegateUpgrade`, `delegateRollback` and `setUseLatestDelegate`.
As well as a few additional view functions that allow the smartnode to query these values. These methods are self explanatory.

The oDAO can introduce an upgraded delegate through the existing proposal system. One member will propose an upgrade to the "rocketMinipoolDelegate" contract and the other members will vote yes or no. Once
a majority is reached the proposal can be executed and the new contract will be adopted. Future `RocketMinipool` deployments will now use this new contract instead. Any minipools with `useLatestDelegate` set
to true, will immediately start using the new contract as any calls to the above fallback function will be directed to the new address. Node operators who have not set `useLatestDelegate` will also be able to 
call `delegateUpgrade` which will update their `rocketMinipoolDelegate` to the new one.
