---
title: "Tick 舍入"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Tick 舍入

我们来看一下，为了支持不同的 tick 间隔，我们还需进行哪些改动。

大于 1 的 tick 间隔将不允许用户选择任意的价格区间：区间的 tick index 必须是 tick 间隔的整数倍。例如，当 tick 间隔为 60 时，我们选择的 tick 可以是：0，60，120，180等等。因此，当用户选择某个区间是，我们需要将其舍入到最近的，满足 tick 间隔整数倍的区间中。

## JavaScript 中的 `nearestUsableTick`
在 [the Uniswap V3 SDK](https://github.com/Uniswap/v3-sdk)中， 实现这个功能的函数叫做 [nearestUsableTick](https://github.com/Uniswap/v3-sdk/blob/b6cd73a71f8f8ec6c40c130564d3aff12c38e693/src/utils/nearestUsableTick.ts):
```javascript
/**
 * Returns the closest tick that is nearest a given tick and usable for the given tick spacing
 * @param tick the target tick
 * @param tickSpacing the spacing of the pool
 */
export function nearestUsableTick(tick: number, tickSpacing: number) {
  invariant(Number.isInteger(tick) && Number.isInteger(tickSpacing), 'INTEGERS')
  invariant(tickSpacing > 0, 'TICK_SPACING')
  invariant(tick >= TickMath.MIN_TICK && tick <= TickMath.MAX_TICK, 'TICK_BOUND')
  const rounded = Math.round(tick / tickSpacing) * tickSpacing
  if (rounded < TickMath.MIN_TICK) return rounded + tickSpacing
  else if (rounded > TickMath.MAX_TICK) return rounded - tickSpacing
  else return rounded
}
```

其核心仅仅是下面这句：
```javascript
Math.round(tick / tickSpacing) * tickSpacing
```

`Math.round` 就是四舍五入到最近整数。

所以，在前端中，构造 `mint` 参数时我们将使用 `nearestUsableTick`：
```javascript
const mintParams = {
  tokenA: pair.token0.address,
  tokenB: pair.token1.address,
  tickSpacing: pair.tickSpacing,
  lowerTick: nearestUsableTick(lowerTick, pair.tickSpacing),
  upperTick: nearestUsableTick(upperTick, pair.tickSpacing),
  amount0Desired, amount1Desired, amount0Min, amount1Min
}
```

> 在实际中，当用户每次调整价格区间的时候它都会被调用，以便于用户看到将会实际创建的价格位置。在我们的简化版实现中，我们并没有做到如此的用户友好。

我们也会在我们的 Solidity 测试中想要用到这样的函数，但是我们用到的所有数学库都没有实现这个功能。

## Solidity 中的 `nearestUsableTick`

在我们的合约测试中，我们也需要一种方式来对于 tick 进行舍入以及把舍入后的价格转换成 $\sqrt{P}$。在前一章中，我们使用了 [ABDKMath64x64](https://github.com/abdk-consulting/abdk-libraries-solidity) 来处理测试中定点数的运算。然而这个库没有实现我们希望对标 `nearestUsableTick` 的函数，所以我们需要自己实现：

```solidity
function divRound(int128 x, int128 y)
    internal
    pure
    returns (int128 result)
{
    int128 quot = ABDKMath64x64.div(x, y);
    result = quot >> 64;

    // Check if remainder is greater than 0.5
    if (quot % 2**64 >= 0x8000000000000000) {
        result += 1;
    }
}
```

这个函数做了这些事情：
1. 对两个 Q64.64 的数做除法；
2. 把结果舍入到十进制整数（`result = quot >> 64`），分数部分被扔掉（向下舍入）；
3. 商除以 $2^{64}$，取余数，并与 `0x8000000000000000` (Q64.64 里的 0.5) 比较；
4. 如果余数大于等于 0.5，把结果向上舍入。

这样得到了一个相当于 JavaScript 中 `Math.round` 的函数，我们就可以实现 `nearestUsableTick` 了：

```solidity
function nearestUsableTick(int24 tick_, uint24 tickSpacing)
    internal
    pure
    returns (int24 result)
{
    result =
        int24(divRound(int128(tick_), int128(int24(tickSpacing)))) *
        int24(tickSpacing);

    if (result < TickMath.MIN_TICK) {
        result += int24(tickSpacing);
    } else if (result > TickMath.MAX_TICK) {
        result -= int24(tickSpacing);
    }
}
```

完成了！