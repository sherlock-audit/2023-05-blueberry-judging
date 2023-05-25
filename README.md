# Issue H-1: BalancerPairOracle will return highly incorrect price if one token isn't 18 dp 

Source: https://github.com/sherlock-audit/2023-05-blueberry-judging/issues/28 

## Found by 
0x52
## Summary

When calculating the fair reserves of the balancer LP, the contract makes the mistake of passing in 18 dp prices but leaving balances in the native decimals of the tokens. This will skew the fair reserves heavily towards one token if it is not 18 dp. The result is that the oracle will return a price that off by a huge amount.

## Vulnerability Detail

See summary.

## Impact

Oracle is unreliable and dangerous if either token isn't 18 dp

## Code Snippet

https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/oracle/BalancerPairOracle.sol#L70-L92

## Tool used

Manual Review

## Recommendation

Adjust the reserves to 18 dp before passing into computeFairReserves.

# Issue H-2: AuraSpell#openPositionFarm fails to return all rewards to user 

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

# Issue H-3: ShortLongSpell#openPosition uses the wrong balanceOf when determining how much collateral to put 

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

# Issue M-1: ShortLongSpell#openPosition attempts to burn wrong token 

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

# Issue M-2: Updating the feeManger on config will cause desync between bank and vaults 

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

