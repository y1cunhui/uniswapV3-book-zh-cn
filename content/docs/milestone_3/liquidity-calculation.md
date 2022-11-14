---
title: "流动性计算"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 流动性计算

在 Uniswap V3 的所有数学公式中，只剩流动性计算我们还没在 Solidity 里面实现。在 Python 脚本中我们有这两个函数：

```python
def liquidity0(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return (amount * (pa * pb) / q96) / (pb - pa)


def liquidity1(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return amount * q96 / (pb - pa)
```

接下来我们在 Solidity 中实现这个计算，以便于在 `Manager.mint()` 中使用。

## 实现 Token X 的流动性计算

我们要实现的函数是在已知 token 数量和价格区间的情况下计算流动性（$L = \sqrt{xy}$）。我们之前已经知道了所有相关的公式，让我们回顾一下：

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

在上一章，我们使用这个函数来计算交易数量（这里的 $\Delta x$），现在我们用它来计算 $L$：

$$L = \frac{\Delta x}{\Delta \frac{1}{\sqrt{P}}}$$

简化之后：
$$L = \frac{\Delta x \sqrt{P_u} \sqrt{P_l}}{\sqrt{P_u} - \sqrt{P_l}}$$

> 这个公式来自于[计算流动性](https://y1cunhui.github.io/uniswapV3-book-zh-cn/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)一章。

在 Solidity 中，我们仍然使用 `PRBMath` 来处理乘除过程中可能出现的溢出：

```solidity
function getLiquidityForAmount0(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    uint256 intermediate = PRBMath.mulDiv(
        sqrtPriceAX96,
        sqrtPriceBX96,
        FixedPoint96.Q96
    );
    liquidity = uint128(
        PRBMath.mulDiv(amount0, intermediate, sqrtPriceBX96 - sqrtPriceAX96)
    );
}
```

## 实现 Token Y 的流动性计算

类似得，我们将使用[计算流动性](https://y1cunhui.github.io/uniswapV3-book-zh-cn/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)中出现的另一个公式来在给定 $y$ 的数量和价格区间的前提下计算流动性：

$$\Delta y = \Delta\sqrt{P} L$$
$$L = \frac{\Delta y}{\sqrt{P_u}-\sqrt{P_l}}$$

```solidity
function getLiquidityForAmount1(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    liquidity = uint128(
        PRBMath.mulDiv(
            amount1,
            FixedPoint96.Q96,
            sqrtPriceBX96 - sqrtPriceAX96
        )
    );
}
```

希望这些代码足够清晰易懂。

## 找到公平的流动性

你可能会问为什么我们有两种方式来计算 $L$，但是我们实际上只有一个 $L$，即 $L = \sqrt{xy}$，究竟哪种方法是正确的？答案是：两种方法都是正确的。

在上面的公式中，我们基于不同的参数来计算$L$：价格区间和其中某种 token 的数量。不同的价格区间和不同的 token 数量会导致不同的 $L$。有一个场景是我们需要计算两个$L$并且从中挑选出一个的，回顾一下之前的 `mint` 函数：

```solidity
if (slot0_.tick < lowerTick) {
    amount0 = Math.calcAmount0Delta(...);
} else if (slot0_.tick < upperTick) {
    amount0 = Math.calcAmount0Delta(...);

    amount1 = Math.calcAmount1Delta(...);

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
} else {
    amount1 = Math.calcAmount1Delta(...);
}
```

在计算流动性时有如下逻辑：
1. 如果我们在一个低于现价的价格区间计算流动性，我们使用 $\Delta y$ 版本的公式；
2. 如果我们在一个高于现价的价格区间计算流动性，我们使用 $\Delta x$ 版本的公式；
3. 如果现价在价格区间内，我们两个都计算并且挑选较小的一个。

> 同样，这些也在[计算流动性](https://y1cunhui.github.io/uniswapV3-book-zh-cn/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)一章中讨论过。

我们现在来实现其中的逻辑。
当现价低于价格区间下界时：

```solidity
function getLiquidityForAmounts(
    uint160 sqrtPriceX96,
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    if (sqrtPriceX96 <= sqrtPriceAX96) {
        liquidity = getLiquidityForAmount0(
            sqrtPriceAX96,
            sqrtPriceBX96,
            amount0
        );
```

当现价位于价格区间中，我们挑选较小的一个：
```solidity
} else if (sqrtPriceX96 <= sqrtPriceBX96) {
    uint128 liquidity0 = getLiquidityForAmount0(
        sqrtPriceX96,
        sqrtPriceBX96,
        amount0
    );
    uint128 liquidity1 = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceX96,
        amount1
    );

    liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
```

最后一种情况：
```solidity
} else {
    liquidity = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceBX96,
        amount1
    );
}
```

Done.