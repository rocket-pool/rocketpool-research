Overview
========

The new simplified withdrawal method removes the requirement for oDAO to submit starting and ending balances. It also removes the requirement for a minipool to be `Withdrawable`
in order to distribute a withdrawal balance. Furthermore, the new method does not rely on any Rocket Pool upgradable contracts to run and can be called by anyone (not just the node operator).
These changes significantly reduce the oDAOs power over the network by limiting their ability to upgrade contracts and direct funds to themselves.

Note: the oDAO still marks minipools as `Withdrawable` and node operators can then finalise a minipool. The purpose for this is that we still need to know if a minipool is staking in order
to issue the correct RPL rewards. The important thing is that ending balances are no longer submitted and the `Withdrawable` status is not required to process a withdrawal.

Methods
=======

There are now two withdrawal methods `distributeBalance()` and `distributeBalanceAndFinalise()` the later just being a convience method that node operators can call to distribute and
finalise the pool in a single transaction.

`distributeBalance()` can be called multiple times but each time is treated as if the entire balance is the full validator withdrawal amount.

The below conditions must be met to call the two relevant withdrawal methods on a minipool:

distributeBalanceAndFinalise()
------------------------------

* Minipool must be `Withdrawable`.
* Can only be called by minipool owner.
* Can only be called once (i.e. when the pool is not already finalised).

distributeBalance()
-------------------

* If minipool is `Staking`
    * Balance must be greater than 16 ETH
* If minipool is `Withdrawable`
    * Can be called by owner; or
    * If called by non-owner
        * Balance must be greater than 16 ETH; or
        * Balance must be greather than 4 ETH and 14 days must have passed since being made `Withdrawable`


Scenarios
=========

Below is a comprehensive list of the potential withdraw scenarios for the new withdrawal process.

Scenario A - More than 32 ETH is returned to the minipool
---------------------------------------------------------
1. The oDAO marks the minipool as `Withdrawable`.
2. NO calls `distributeBalanceAndFinalise()`.
3. User funds are returned to rETH contract + share of rewards - node commission.
4. NO funds are returned + share of rewards + node commission.
5. NO's minipool count is decremented so they can now withdraw their RPL stake.

Scenario B - More than 32 ETH is returned to the contract but the oDAO is not functioning
-----------------------------------------------------------------------------------------
1. oDAO does not mark pool as `Withdrawable` so the minipool remains in `Staking` status.
2. NO can still call `distributeBalance()` as it does not interact with any network contracts or `RocketStorage`.
3. User funds are returned to rETH contract + share of rewards - node commission.
4. NO funds are returned + share of rewards + node commission.
5. NO cannot unstake their RPL because their minipool count has not been decremented by finalisation.

> Note: NO cannot call `distributeBalanceAndFinalise()` because it requires minipool to be in `Withdrawable` state. 
> It also interacts with other network contracts which may have been maliciously upgraded. However, if there is a 
> malicious contract or inoperable oDAO than RPL will have lost it's value and there is no need to finalise the pool 
> to recover RPL stake. The only useful course of action is to recover ETH which is what this method does.

Scenario C - Less than 32 ETH but more than 16 ETH is returned
---------------------------------------------------------------
1. oDAO marks pool as `Withdrawable`.
2. NO calls `distributeBalanceAndFinalise()`.
3. 16 ETH is returned to rETH contract.
4. Remaining ETH is returned to NO.
5. NO can unstake their RPL normally.

> Note: works as long as NO still receives enough ETH to make it worth the gas. If not, see Scenario D

Scenario D - Less than 32 ETH but more than 16 ETH is returned and NO won't distribute balance
----------------------------------------------------------------------------------------------
1. oDAO marks pool as `Withdrawable`.
2. Anyone can call `distributeBalance()` because minipool is `Withdrawable` and balance is > 16 ETH.
3. 16 ETH is returned to rETH contract.
4. Remaining ETH is returned to NO.
5. NO can call `finalise()` if they want to finalise to pool to unlock their RPL stake.

Scenario E - Less than 16 ETH is returned
-----------------------------------------
1. oDAO marks pool as `Withdrawable`.
2. NO wants their RPL stake back so they are forced to call `distributeBalanceAndFinalise()` to free up their stake.
3. Total balance is sent to rETH contract.
4. NO's RPL is slashed and auctioned in an attempt to make up the loss.
5. NO can unstake their RPL normally.

> Note: unless the NO's RPL is worth a lot of value, it is unlikely they will call this method. If so, see Scenario F.

Scenario F - Less than 16 ETH but more than 4 ETH is returned and NO won't distribute balance
---------------------------------------------------------------------------------------------
1. oDAO marks pool as `Withdrawable`.
2. After 14 days pass, anyone can now call `distributeBalance()`.
3. Appropriate amount of RPL to slash is recorded on minipool contract.
4. Total balance is sent to rETH contract.
5. If NO wants to withdraw their RPL stake they must call `finalise()` which slashes their RPL and auctions it to recoup loss.
6. Alternatively, if the incentive is not there for them to do so, anyone can call `slash()` to execute the slashing on behalf of the network.

> Note: `distributeBalance()` purposefully doesn't execute the slash method as this method reads from RocketStorage. 
> The two methods have been kept separate so that processing withdrawals do not rely on RocketStorage or any other network contract.
> Note: The 14 day delay is to prevent public from calling `distributeBalance()` if the oDAO has set the minipool to 
> withdrawable but funds have not been withdrawn yet. 7-10 days. It also allows a NO enough time to withdraw if a
> malicious oDAO marks their pool as withdrawable before it actually is.

Scenario G - Less than 16 ETH but more than 4 ETH is returned and oDAO is not functioning
-----------------------------------------------------------------------------------------
1. Less than 16 ETH but more than 4 ETH is returned to minipool.
2. oDAO does not mark pool as `Withdrawable`.
3. NO can call `distributeBalance()`.
4. Total balance is sent to rETH contract.

> Note: This scenario fails because there is no incentive for NO to call `distributeBalance()` and we can't just allow 
> anyone to call `distributeBalance()` if the pool is still `Staking` and there is less than 16 ETH because that would 
> allow anyone to grief NOs by slashing them. This is worst case scenario and results in loss of funds but should be 
> exceptionally unlikely as slashings greater than 16 ETH will already be incredibly unlikely and then top it off with 
> a malicious oDAO.

Scenario H - Less than 4 ETH is returned
----------------------------------------
1. Less than 4 ETH is returned to the minipool.
2. Only the NO can call `distributeBalance()` or `distributeBalanceAndFinalise()`.

> Note: There is no incentive for them to do so, so funds will be lost in this scenario. However, it is incredibly unlikely
