---
title: "闪电贷"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## 闪电贷

Uniswap V2 和 V3 都实现了闪电贷：不受限且无抵押的借贷，但是必须在同一笔交易中还款。池子可以给予用户请求的任意数量的 token，但是在调用结束时，这笔数量必须被还回，并收取一小笔费用。

闪电贷必须在同一笔交易中偿还的要求意味着通常的用户不能使用闪电贷：因为用户没有办法在交易中实现复杂的逻辑。闪电贷仅能够被智能合约使用和偿还。

闪电贷是 DeFi 中一个非常强有力的工具。尽管它经常被用在攻击 DeFi 协议的漏洞中（通过闪电贷来改变池子余额并且滥用有缺陷的状态管理函数），它同样也有许多正面的应用场景（例如借贷协议中的杠杆头寸管理）——这也是为什么存储流动性的 DeFi 应用通常会提供无需许可的闪电贷接口欧。

### 实现闪电贷

在 Uniswap V2 中，闪电贷是交易函数的一个功能：可以直接在 swap 过程中借出一笔 token，但是你必须在同一笔交易中还回这些 token 或者等价值的另一种池子 token。在 V3 中，闪电贷从交易中分离出来——它变成了一个单独的函数，给调用者一笔他们请求数量的 token，调用他们的 callback 函数，并且确保闪电贷被偿还：

```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(data);

    require(IERC20(token0).balanceOf(address(this)) >= balance0Before);
    require(IERC20(token1).balanceOf(address(this)) >= balance1Before);

    emit Flash(msg.sender, amount0, amount1);
}
```

这个函数把 token 发送给调用者，并且调用它的 `uniswapV3FlashCallback` 函数——调用者需要实现这个函数来偿还贷款。然后函数确保余额并没有降低。注意到，允许向 callback 函数传一些data。

下面是一个 callback 函数实现的样例

```solidity
function uniswapV3FlashCallback(bytes calldata data) public {
    (uint256 amount0, uint256 amount1) = abi.decode(
        data,
        (uint256, uint256)
    );

    if (amount0 > 0) token0.transfer(msg.sender, amount0);
    if (amount1 > 0) token1.transfer(msg.sender, amount1);
}
```

在这个实现中，我们仅仅把 token 还回给池子（在 `flash` 函数测试中使用了这个 callback）。在实际中，它通常会使用贷来的金额去在其他 DeFi 协议上进行一些操作。但是它必须在 callback 函数中偿还贷款。

完成了！