---
title: "用户界面"
weight: 8
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 用户界面

现在，我们可以针对我们在这个 milestone 中做出的改变来更新我们的UI了。我们会增加两个新的特性：
1. 添加流动性对话框
2. 交易时的滑点设置

## 添加流动性对话框

![Add Liquidity dialog window](/images/milestone_3/add_liquidity_dialog.png)

这一步的改变会最终移除代码中硬编码的流动性数量，并且允许我们在任意价格区间添加流动性。

对话框就是一个带一系列输入的简单组件。我们认识可以重用之前实现的 `addLiquidity` 函数。然而，我们现在需要在JavaScript中将价格转换成 tick：我们希望用户输入的是价格信息，但是合约想要的参数是 tick。为了简化我们的实现，我们会使用 [Uniswap V3 官方 SDK](https://github.com/Uniswap/v3-sdk/)来完成这一步。

要把价格转换成 $\sqrt{P}$，我们可以使用 [encodeSqrtRatioX96](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/encodeSqrtRatioX96.ts) 函数。这个函数把两个 token 数量作为输入，计算出对应的价格。由于我们只希望把价格转换成 $\sqrt{P}$，我们给 `amount0` 传参数 1 即可：

```javascript
const priceToSqrtP = (price) => encodeSqrtRatioX96(price, 1);
```

为了把价格转成 tick index，我们可以使用 [TickMath.getTickAtSqrtRatio](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/tickMath.ts#L82) 函数。这是 Solidity 中的 TickMath 库的 JavaScript 版本：


```javascript
const priceToTick = (price) => TickMath.getTickAtSqrtRatio(priceToSqrtP(price));
```

我们现在可以直接把用户输入的价格转换成 tick 了：

```javascript
const lowerTick = priceToTick(lowerPrice);
const upperTick = priceToTick(upperPrice);
```

另一个我们需要添加的点就是滑点保护。为了简化起见，我们在这里把它硬编码为 0.5%。下面的代码展示了如何用滑点来计算最小数量：

```javascript
const slippage = 0.5;
const amount0Desired = ethers.utils.parseEther(amount0);
const amount1Desired = ethers.utils.parseEther(amount1);
const amount0Min = amount0Desired.mul((100 - slippage) * 100).div(10000);
const amount1Min = amount1Desired.mul((100 - slippage) * 100).div(10000);
```

## 交易中的滑点保护

尽管我们是我们系统的唯一用户因此永远不会遇到滑点的问题，我们还是要增加一个输入项来对于交易过程进行滑点保护。

![Main screen of the web app](/images/milestone_3/slippage_tolerance.png)

在交易中，滑点保护是通过限制价格来实现的——我们在交易过程中不会超过或者低于这个价格。这意味着我们需要在发送交易之前就知道价格。然而我们并不需要在前端对此进行计算，因为我们已经在报价合约中实现了相关功能：

```solidity
function quote(QuoteParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) { ... }
```

我们调用它来获取交易数额。

因此，为了计算限制价格，我们使用 `sqrtPriceX96After`，并从其中减去滑点价格——就得到了我们希望在交易中不低于的某个价格。

```solidity
const limitPrice = priceAfter.mul((100 - parseFloat(slippage)) * 100).div(10000);
```

完成了！