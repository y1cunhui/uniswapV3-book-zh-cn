---
title: "简介"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 第一笔交易

在本章中，我们将会搭建一个流动性池合约，它能够接受用户的流动性并且在某个价格区间内做交易。简单起见，我们仅在一个价格区间内提供流动性，并且仅允许单向的交易。另外，为了更好地理解其中的数学原理，我们将手动计算其中用到的数学参数，暂不使用 Solidity 的数学库进行计算。

我们本章中要搭建的模型如下：
1. 这是一个 ETH/USDC 的池子合约。ETH是资产 $x$，USDC是资产 $y$；
2. 现货价格将被设置为一个 ETH 对 5000 USDC；
3. 我们提供流动性的价格区间为一个ETH对 4545 - 5500 USDC；
4. 我们将会从池子中购买 ETH，并且保证价格在上述价格区间内。

模型的图像大致如下：


![Buy ETH for USDC visualization](/images/milestone_1/buy_eth_model.png)



在开始代码部分之前，我们首先会手动计算模型中用到的所有数学参数。简单起见，我将使用 Python 来进行计算而不是 Solidity，因为 Solidity 在数学计算上有很多细微之处需要考虑。因此在这章，我们将会把所有的参数硬编码进池子合约里。这会让我们获得一个最小可用的产品。

本章中所有用到的 python 计算都在 [unimath.py](https://github.com/Jeiwan/uniswapv3-code/blob/main/unimath.py)。

> 你可以在[这个 Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1) 找到本章完整代码.

> 如果你有任何问题，欢迎在[本章的 Github Discussion 中提问和交流](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-1-first-swap)!

