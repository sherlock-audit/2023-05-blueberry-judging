Bauer

high

# AuraSpell `Vault.exitPool` without any slippage protection

## Summary
When users exit the pool from a Balancer pool with minAmountsOut set to 0, they may be vulnerable to sandwich attacks
## Vulnerability Detail
The `AuraSpell.closePositionFarm()` function is used for closing a leveraged position. Inside the function,the protocol remove liquidity from a Balancer pool and expects to receive at least the minimum amount of tokens specified in minAmountsOut. However ,the minAmountsOut is [0,0],it is vulnerable to sandwich attacks.

```solidity
 uint[] memory minAmountsOut = new uint[](2);
            wAuraPools.getVault(lpToken).exitPool(
                IBalancerPool(lpToken).getPoolId(),
                address(this),
                address(this),
                IBalancerVault.ExitPoolRequest(tokens, minAmountsOut, "", false)
            );

```
As the code below, the Vault protocol will check the minimum amount of tokens the user expects to get out of the Pool.
```solidity
 function _processExitPoolTransfers(
        address payable recipient,
        PoolBalanceChange memory change,
        bytes32[] memory balances,
        uint256[] memory amountsOut,
        uint256[] memory dueProtocolFeeAmounts
    ) private returns (bytes32[] memory finalBalances) {
        finalBalances = new bytes32[](balances.length);
        for (uint256 i = 0; i < change.assets.length; ++i) {
            uint256 amountOut = amountsOut[i];
            _require(amountOut >= change.limits[i], Errors.EXIT_BELOW_MIN);
```


## Impact
Users may be vulnerable to sandwich attacks when exiting the pool
## Code Snippet
https://github.com/sherlock-audit/2023-05-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L183-L190
https://dashboard.tenderly.co/tx/mainnet/0x81e61ccf1e4190dd81cc73bbaac497d53d367e4c5804790fa168044df578d525/debugger?trace=0.2
## Tool used

Manual Review

## Recommendation
Set the minimum amount of tokens the user expects to get out of the Pool
