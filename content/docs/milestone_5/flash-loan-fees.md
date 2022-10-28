---
title: "闪电贷费率"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 闪电贷费率

在前一个 milestone 中，我们实现了闪电贷，并且它是免费的。然而，Uniswap 会对闪电贷收取一定的交易费用，我们将会在这里把它加入到我们的实现中：闪电贷返还的数量必须包含一个费用。

更新后的 `flash` 函数长这样：

```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 fee0 = Math.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = Math.mulDivRoundingUp(amount1, fee, 1e6);

    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(
        fee0,
        fee1,
        data
    );

    if (IERC20(token0).balanceOf(address(this)) < balance0Before + fee0)
        revert FlashLoanNotPaid();
    if (IERC20(token1).balanceOf(address(this)) < balance1Before + fee1)
        revert FlashLoanNotPaid();

    emit Flash(msg.sender, amount0, amount1);
}
```

改变的点在于，我们现在根据用户请求的 token 数量来收取费用，并且在还款后检查数量增加了小费的数量。
