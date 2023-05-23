0x52

medium

# Malicious user can permanently DOS collateral on BlueBerryBank by repaying bToken directly

## Summary

bTokens are a fork of Compound's cToken which allows anyone to repay on another's behalf. This can be abused to cause a division by zero error which would DOS the collateral.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-blueberry/blob/96eb1829571dc46e1a387985bd56989702c5e1dc/blueberry-core/contracts/BlueBerryBank.sol#L700-L704

        uint256 totalShare = bank.totalShare;
        uint256 totalDebt = _borrowBalanceStored(token);
        uint256 share = totalShare == 0
            ? amount
            : (amount * totalShare).divCeil(totalDebt);

When borrowing from BlueBerryBank the contract attempts to divide by the totalDebt.

https://github.com/sherlock-audit/2023-04-blueberry/blob/96eb1829571dc46e1a387985bd56989702c5e1dc/blueberry-core/contracts/BlueBerryBank.sol#L320-L324

    function _borrowBalanceStored(
        address token
    ) internal view returns (uint256) {
        return ICErc20(banks[token].bToken).borrowBalanceStored(address(this));
    }

totalDebt is simply the borrowed balance of the bank as reported by the bToken. This is problematic as the banks debt can be paid off directly to the bToken. Once repaid down to zero the calculation would always throw an error and revert.

## Impact

Malicious user can permanently DOS bank collateral 

## Code Snippet

https://github.com/sherlock-audit/2023-04-blueberry/blob/96eb1829571dc46e1a387985bd56989702c5e1dc/blueberry-core/contracts/BlueBerryBank.sol#L679-L713

## Tool used

Manual Review

## Recommendation

This is a tricky edge case and I don't see any straight forward way to mitigate it