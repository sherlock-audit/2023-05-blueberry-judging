# Issue H-1: AuraSpell#openPositionFarm fails to return all rewards to user 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/29 

## Found by 
0x52, nobody2018
## Summary

When a user adds to an existing position on AuraSpell, the contract burns their current position and remints them a new one. The issues is that WAuraPool will send all reward tokens to the contract but it only sends Aura back to the user, causing all other rewards to be lost.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L256-L261

        for (uint i = 0; i < rewardTokens.length; i++) {
            IERC20Upgradeable(rewardTokens[i]).safeTransfer(
                msg.sender,
                rewards[i]
            );
        }

Inside WAuraPools#burn reward tokens are sent to the user.

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L130-L140

        IBank.Position memory pos = bank.getCurrentPositionInfo();
        if (pos.collateralSize > 0) {
            (uint256 pid, ) = wAuraPools.decodeId(pos.collId);
            if (param.farmingPoolId != pid)
                revert Errors.INCORRECT_PID(param.farmingPoolId);
            if (pos.collToken != address(wAuraPools))
                revert Errors.INCORRECT_COLTOKEN(pos.collToken);
            bank.takeCollateral(pos.collateralSize);
            wAuraPools.burn(pos.collId, pos.collateralSize);
            _doRefundRewards(AURA);
        }

We see above that the contract only refunds Aura to the user causing all other extra reward tokens received by the contract to be lost to the user.

## Impact

User will lose all extra reward tokens from their original position

## Code Snippet

## Tool used

Manual Review

## Recommendation

WAuraPool returns the reward tokens it sends. Use this list to refund all tokens to the user



## Discussion

**sleepriverfish**

Escalate for 10 USDC.
The issue was  excluded from #Blueberry Update, it appears to have been rewarded in #Blueberry Update 2.
https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/42

**sherlock-admin**

 > Escalate for 10 USDC.
> The issue was  excluded from #Blueberry Update, it appears to have been rewarded in #Blueberry Update 2.
> https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/42

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**securitygrid**

Escalate for 10 USDC
valid H. Nobody escalated [it](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/42) 

**sherlock-admin**

 > Escalate for 10 USDC
> valid H. Nobody escalated [it](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/42) 

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**IAm0x52**

Agreed with second escalation

**sleepriverfish**

So, the issue https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/42 considered invalid? I believe it should be categorized and rewarded in some way.

**hrishibhat**

Escalation rejected

Valid high
The issue mentioned above has been resolved accordingly in the respective contest. 

**sherlock-admin**

> Escalation rejected
> 
> Valid high
> The issue mentioned above has been resolved accordingly in the respective contest. 

    This issue's escalations have been rejected!

    Watsons who escalated this issue will have their escalation amount deducted from their next payout.

# Issue H-2: ShortLongSpell#openPosition uses the wrong balanceOf when determining how much collateral to put 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/31 

## Found by 
0x52
## Summary

The _doPutCollateral subcall in ShortLongSpell#openPosition uses the balance of the uToken rather than the vault resulting in the vault tokens being left in the contract which will be stolen.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/ShortLongSpell.sol#L144-L150

        address vault = strategies[param.strategyId].vault;
        _doPutCollateral(
            vault,
            IERC20Upgradeable(ISoftVault(vault).uToken()).balanceOf(
                address(this)
            )
        );

When putting the collateral the contract is putting vault but it uses the balance of the uToken instead of the balance of the vault.

## Impact

Vault tokens will be left in contract and stolen

## Code Snippet

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/ShortLongSpell.sol#L111-L151

## Tool used

Manual Review

## Recommendation

Use the balanceOf vault rather than vault.uToken



## Discussion

**sleepriverfish**

Escalate for 10 USDC

In #Blueberry Update, despite the successful escalation of the issue, no reward was granted for the heightened severity and impact of the vulnerability. However, in #Blueberry Update2, a reward was offered specifically for the detection and reporting of a similar vulnerability.
https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/116

**sherlock-admin**

 > Escalate for 10 USDC
> 
> In #Blueberry Update, despite the successful escalation of the issue, no reward was granted for the heightened severity and impact of the vulnerability. However, in #Blueberry Update2, a reward was offered specifically for the detection and reporting of a similar vulnerability.
> https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/116

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**IAm0x52**

This issue has been escalated in the first contest and should be valid in both

**hrishibhat**

Escalation accepted

This is a valid issue, and necessary decisions have been taken in the respective issue from the previous contest. 


**sherlock-admin**

> Escalation accepted
> 
> This is a valid issue, and necessary decisions have been taken in the respective issue from the previous contest. 
> 

    This issue's escalations have been accepted!

    Contestants' payouts and scores will be updated according to the changes made on this issue.

# Issue M-1: BalancerPairOracle#getPrice will revert due to division by zero in some cases 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/25 

## Found by 
0x52, nobody2018
## Summary

`BalancerPairOracle#getPrice` internally calls `computeFairReserves`, which returns fair reserve amounts given spot reserves, weights, and fair prices. When the parameter `resA` passed to `computeFairReserves` is smaller than `resB`, division by 0 will occur.

## Vulnerability Detail

In `BalancerPairOracle#getPrice`, resA and resB passed to `computeFairReserves` are the balance of TokenA and TokenB of the pool respectively. It is common for the balance of TokenB to be greater than the balance of TokenA.

```solidity
function computeFairReserves(
        uint256 resA,
        uint256 resB,
        uint256 wA,
        uint256 wB,
        uint256 pxA,
        uint256 pxB
    ) internal pure returns (uint256 fairResA, uint256 fairResB) {
    	...
    	//@audit r0 = 0 when resA < resB.
->      uint256 r0 = resA / resB;
        uint256 r1 = (wA * pxB) / (wB * pxA);
        // fairResA = resA * (r1 / r0) ^ wB
        // fairResB = resB * (r0 / r1) ^ wA
        if (r0 > r1) {
            uint256 ratio = r1 / r0;
            fairResA = resA * (ratio ** wB);
            fairResB = resB / (ratio ** wA);
        } else {
->          uint256 ratio = r0 / r1;		// radio = 0 when r0 = 0
->          fairResA = resA / (ratio ** wB);   	// revert divided by 0
            fairResB = resB * (ratio ** wA);
        }
    }
```

Another case is **when the decimals of tokenA is smaller than the decimals of tokenB**, such as usdc(e6)-weth(e18).

## Impact

All functions that subcall `BalancerPairOracle#getPrice` will be affected.

## Code Snippet

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/oracle/BalancerPairOracle.sol#L35-L66

## Tool used

Manual Review

## Recommendation

```diff
--- a/blueberry-core/contracts/oracle/BalancerPairOracle.sol
+++ b/blueberry-core/contracts/oracle/BalancerPairOracle.sol
@@ -50,7 +50,7 @@ contract BalancerPairOracle is UsingBaseOracle, IBaseOracle {
         // --> fairResA / r1^wB = constant product
         // --> fairResA = resA^wA * resB^wB * r1^wB
         // --> fairResA = resA * (resB/resA)^wB * r1^wB = resA * (r1/r0)^wB
-        uint256 r0 = resA / resB;
+        uint256 r0 = resA * 10**(decimalsB) / resB;
         uint256 r1 = (wA * pxB) / (wB * pxA);
         // fairResA = resA * (r1 / r0) ^ wB
         // fairResB = resB * (r0 / r1) ^ wA
```



## Discussion

**securitygrid**

Escalate for 10 USDC
This is a valid M/H. Please review **Vulnerability Detail** of this report, it describes 2 cases:
1. resA and resB passed to computeFairReserves are the balance of TokenA and TokenB of the pool respectively. It is common for the balance of TokenB to be greater than the balance of TokenA.
2. when the decimals of tokenA is smaller than the decimals of tokenB, such as usdc(e6)-weth(e18).

**This issue is same root as #28**.
The impact described in #28 is only possible if the dp of the two tokens are different. But it is not completely correct, because when the dp of token0 is less than the dp of token1, a division by zero error occurs, such as token0(e6)-token1(e18).
Merging the two reports is the best description.

**sherlock-admin**

 > Escalate for 10 USDC
> This is a valid M/H. Please review **Vulnerability Detail** of this report, it describes 2 cases:
> 1. resA and resB passed to computeFairReserves are the balance of TokenA and TokenB of the pool respectively. It is common for the balance of TokenB to be greater than the balance of TokenA.
> 2. when the decimals of tokenA is smaller than the decimals of tokenB, such as usdc(e6)-weth(e18).
> 
> **This issue is same root as #28**.
> The impact described in #28 is only possible if the dp of the two tokens are different. But it is not completely correct, because when the dp of token0 is less than the dp of token1, a division by zero error occurs, such as token0(e6)-token1(e18).
> Merging the two reports is the best description.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**IAm0x52**

Agreed with escalation. This and #28 are dupes and this does a better job of describing the issue so it should be the main issue. Additionally given that the contract would become nonfunctional rather than return an incorrect price, I agree with the watson's original severity of medium.

**hrishibhat**

Escalation accepted

Valid medium
Making this issue the main one and #28 a duplicate of this issue. 


**sherlock-admin**

> Escalation accepted
> 
> Valid medium
> Making this issue the main one and #28 a duplicate of this issue. 
> 

    This issue's escalations have been accepted!

    Contestants' payouts and scores will be updated according to the changes made on this issue.

# Issue M-2: ShortLongSpell#openPosition attempts to burn wrong token 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/30 

## Found by 
0x52
## Summary

ShortLongSpell#openPosition attempts to burn vault.uToken when it should be using vault instead. The result is that ShortLongSpell#openPosition will be completely nonfunctional when the user is adding to their position

## Vulnerability Detail

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/ShortLongSpell.sol#L133-L140

            address burnToken = address(ISoftVault(strategy.vault).uToken());
            if (collSize > 0) {
                if (posCollToken != address(wrapper))
                    revert Errors.INCORRECT_COLTOKEN(posCollToken);
                bank.takeCollateral(collSize);
                wrapper.burn(burnToken, collSize);
                _doRefund(burnToken);
            }

We see above that the contract attempts to withdraw vault.uToken from the wrapper.

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/ShortLongSpell.sol#L145-L150

        _doPutCollateral(
            vault,
            IERC20Upgradeable(ISoftVault(vault).uToken()).balanceOf(
                address(this)
            )
        );

This is in direct conflict with the collateral that is actually deposited which is vault. This will cause the function to always revert when adding to an existing position.

## Impact

ShortLongSpell#openPosition will be completely nonfunctional when the user is adding to their position

## Code Snippet

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/ShortLongSpell.sol#L111-L151

## Tool used

Manual Review

## Recommendation

Burn token should be vault rather than vault.uToken

# Issue M-3: Updating the feeManger on config will cause desync between bank and vaults 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/32 

## Found by 
0x52
## Summary

When the bank is initialized it caches the current config.feeManager. This is problematic since feeManger can be updated in config. Since it is precached the address in bank will not be updated leading to a desync between contracts the always pull the freshest value for feeManger and bank.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/BlueBerryBank.sol#L142

        feeManager = config_.feeManager();

Above we see that feeManger is cached during initialization.

 https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/vault/HardVault.sol#L140-L143

        withdrawAmount = config.feeManager().doCutVaultWithdrawFee(
            address(uToken),
            shareAmount
        );

This is in direct conflict with other contracts the always use the freshest value. This is problematic for a few reasons. The desync will lead to inconsistent fees across the ecosystem either charging users too many fees or not enough.

## Impact

After update users will experience inconsistent fees across the ecosystem

## Code Snippet


https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/BlueBerryBank.sol#L142

## Tool used

Manual Review

## Recommendation

BlueBerryBank should always use config.feeManger instead of caching it.

