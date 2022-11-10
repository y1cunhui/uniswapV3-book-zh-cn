---
title: "通用mint"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 通用 mint

现在，我们可以更新 `mint` 函数来直接在 Solidity 中进行计算而不需要手动计算并硬编码了。

## 初始化 tick 与更新

还记得在 `mint` 函数中，我们更新 TickInfo 这个 mapping 来存储 tick 中可用的流动性信息。现在，我们将会使用新的 bitmap 索引来进行这一步——我们之后会用这个新的索引来在交易中寻找下一个可用 tick。

首先，我们需要更新 `Tick.update` 函数：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal returns (bool flipped) {
    ...
    flipped = (liquidityAfter == 0) != (liquidityBefore == 0);
    ...
}
```

现在，它会返回一个 `flipped` flag，当流动性被添加到一个空的 tick 或整个 tick 的流动性被耗尽时为 true。

接下来，在 `mint` 函数中，我们更新 bitmap 索引：

```solidity
// src/UniswapV3Pool.sol
...
bool flippedLower = ticks.update(lowerTick, amount);
bool flippedUpper = ticks.update(upperTick, amount);

if (flippedLower) {
    tickBitmap.flipTick(lowerTick, 1);
}

if (flippedUpper) {
    tickBitmap.flipTick(upperTick, 1);
}
...
```

> 再次说明，在 Milestone 4 之前，TickSpacing 参数的值会始终为1.

## Token 数量计算

`mint` 函数中最大的变化就是 token 数量的计算。在 milestone 1 中，我们硬编码了这些值：

```solidity
    amount0 = 0.998976618347425280 ether;
    amount1 = 5000 ether;
```

现在，我们将使用与 milestone 1 中相同的公式，在 Solidity 中计算它。回顾一下这些公式：

$$\Delta x = \frac{L(\sqrt{p(i_u)} - \sqrt{p(i_c)})}{\sqrt{p(i_u)}\sqrt{p(i_c)}}$$
$$\Delta y = L(\sqrt{p(i_c)} - \sqrt{p(i_l)})$$

$\Delta x$ 代表 `token0` 的数量, 即 token $x$。让我们在 Solidity 中进行实现：
```solidity
// src/lib/Math.sol
function calcAmount0Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount0) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    require(sqrtPriceAX96 > 0);

    amount0 = divRoundingUp(
        mulDivRoundingUp(
            (uint256(liquidity) << FixedPoint96.RESOLUTION),
            (sqrtPriceBX96 - sqrtPriceAX96),
            sqrtPriceBX96
        ),
        sqrtPriceAX96
    );
}
```
> 这个函数的功能与 Python 脚本中的 `calc_amount0` 一致。

第一步是将两个价格排序来保证减法时不会溢出。接下来，我们将 `liquidity` 转换成 Q96.64 格式的数字，只需要乘以 2**96。下一步，根据公式，我们将其乘以价格之差并除以两个价格（先除以大的，再除以小的）。两个除法的顺序并不重要，但是我们的除法要分两步进行，因为分母的乘法可能会导致溢出。

我们使用了 `mulDivRoundingUp` 函数来在一步中进行乘除。这个函数是基于 `PRBMath` 库中的 `mulDiv`：

```solidity
function mulDivRoundingUp(
    uint256 a,
    uint256 b,
    uint256 denominator
) internal pure returns (uint256 result) {
    result = PRBMath.mulDiv(a, b, denominator);
    if (mulmod(a, b, denominator) > 0) {
        require(result < type(uint256).max);
        result++;
    }
}
```

`mulmod` 是Solidity的一个函数，将两个数 `a` 和 `b` 相乘，乘积除以 `denominator`，返回余数。如果余数为正，我们将结果上取整。

接下来是 $\Delta y$：
```solidity
function calcAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount1) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    amount1 = mulDivRoundingUp(
        liquidity,
        (sqrtPriceBX96 - sqrtPriceAX96),
        FixedPoint96.Q96
    );
}
```
> 这个函数与 Python 脚本中的 `calc_amount1` 一致。

同样我们使用 `mulDivRoundingUp` 来防止乘法过程中的溢出。

现在它们都完成了！我们现在可以使用这些函数来计算 token 数量了：

```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    Slot0 memory slot0_ = slot0;

    amount0 = Math.calcAmount0Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );

    amount1 = Math.calcAmount1Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(lowerTick),
        amount
    );
    ...
}
```

其余的一切都保持不变。测试脚本中的数字需要更新，因为取整的缘故会与我们一开始手动计算的略有不同。
