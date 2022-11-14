---
title: "不同价格区间"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 不同价格区间

在我们之前的实现中，我们仅仅创建包含现价的价格区间：

```solidity
// src/UniswapV3Pool.sol
function mint(
    ...
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

    liquidity += uint128(amount);
    ...
}
```

从这段代码中你也可以看到，我们总是会更新流动性 tracker（跟踪现在可用的流动性，即在现价时候可用的流动性）。

然而，在现实中，也可以创建**低于或高于**现价的价格区间。Uniswap V3 的设计允许 LP 提供当前不可用的流动性。这些流动性只有当现价进入这些“休眠”的流动性区间时才会被“激活”。

以下是几种可能存在的价格区间情况：
1. 活跃价格区间，也即包含现价的价格区间；
2. 低于现价的价格区间，该价格区间的上界 tick 低于现价 tick；
3. 高于现价的价格区间，该价格区间的下界 tick 高于现价 tick。

## 限价单

一个有趣的事实是：非活跃的流动性（所在区间不包含现价）可以被看做是*限价单(limit orders)*。

在交易中，限价单是一种当价格突破用户设定的某个点的时候才会被执行的订单。例如，你希望开一个限价单，当 ETH 价格低于 $1000 的时候买入一个 ETH。类似地，你可以使用限价单来出售资产。在Uniswap V3 中，你可以通过在非活跃价格区间提供流动性来达到类似的目的。让我们来看一下它如何工作：

![Liquidity ranges outside of the current price](/images/milestone_3/ranges_outside_current_price.png)

如果你在低于或高于现价的位置提供流动性（整个价格区间都低于/高于现价），那么你提供的流动性将完全由**一种资产**组成——两种资产中较便宜的那一种。在我们的例子中，我们的池子是把ETH作为 token $x$，把USDC作为 token $y$ ，我们的价格定义为：

$$P = \frac{y}{x}$$

如果我们把流动性放置在低于现价的区间，那么流动性将只会由 USDC 组成，因为当我们添加流动性的时候 USDC 的价格低于现价。类似地，如果我们高于现价的区间提供流动性，那么流动性将完全由 ETH 组成，因为 ETH 的价格低于现价。

回顾一下我们在简介中的图表：

![Price range depletion](/images/milestone_1/range_depleted.png)

如果我们购买这个区间中所有可用的 ETH，这个区间内将只会由另一种 token 组成，USDC，并且价格将会沿着曲线移动到最右边。这个价格，也即 $\frac{y}{x}$，会**升高**。如果有一个价格区间在当前区间的右边，它将需要提供 ETH 的流动性，并且仅包含 ETH：它需要为接下来的交易提供 ETH。如果我们持续购买并且拉高价格，我们可能会继续“耗尽”下一个价格区间，也即买走它所有的 ETH 并卖出 USDC。同样，这个区间会以全部是 USDC 而中止，现价移出这个区间。

类似地，如果我们购买 USDC，我们会使得价格向左移动并且从池子中移出 USDC。下一个价格区间会仅包含 USDC 代币来满足需求，并且类似地，如果我们继续买光这个区间的所有 USDC，它也会以仅包含 ETH 而中止。

注意到一个有趣的点：当跨越整个价格区间时，其中的流动性从一种 token 交易为另外一种。并且如果我们设置一个相当窄的价格区间，价格会快速越过整个区间，我们就得到了一个限价单！例如，如果我们想在某个低价点购入 ETH，我们可以在一个低价区间提供仅包含 USDC 的流动性，等待价格越过这个区间。在这之后，我们就可以移出流动性，所有的 USDC 都转换成了 ETH！

希望这个例子没有让你感到困惑，我觉得这是一个理解价格区间动态变化的很好的例子。

## 更新 `mint` 函数

为了支持上面提到的各种价格区间，我们需要知道现价究竟是低于、位于，还是高于用户提供的价格区间，并且计算相应的 token 数量。如果价格区间高于现价，我们希望它的流动性仅仅由 token $x$ 组成：

```solidity
// src/UniswapV3Pool.sol
function mint(
    ...
    if (slot0_.tick < lowerTick) {
        amount0 = Math.calcAmount0Delta(
            TickMath.getSqrtRatioAtTick(lowerTick),
            TickMath.getSqrtRatioAtTick(upperTick),
            amount
        );
    ...
```
当价格区间包含现价，我们希望两种 token 的价格与现价成比例（这是我们之前实现的部分）：

```solidity
} else if (slot0_.tick < upperTick) {
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

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
```
注意到，这是唯一一个我们会更新 `liquidity` 的场景，因为这个变量是跟踪现在可用的流动性数量。

当价格区间低于现价，区间仅仅包含 token $y$：

```solidity
} else {
    amount1 = Math.calcAmount1Delta(
        TickMath.getSqrtRatioAtTick(lowerTick),
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );
}
```

这就是 `mint` 的全部更新！