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

在本章中，我们将会搭建一个流动性池合约，能够接受用户的流动性并且在某个价格区间内做交易。为了尽可能简化，我们仅在一个价格区间内提供流动性，并且仅允许单向的交易。另外，为了更好地理解其中的数学原理，我们将手动计算其中用到的数学参数，暂不使用Solidity的数学库进行计算。

我们本章中要搭建的模型如下：
1. 这是一个ETH/USDC的池子合约。ETH是资产$x$，USDC是资产$y$。
2. 现货价格将被设置为一个ETH对5000USDC
3. 我们提供流动性的价格区间为一个ETH对4545-5500USDC
4. 我们将会从池子中购买ETH，并且保证价格在上述价格区间内。

模型的图像大致如下：

![Buy ETH for USDC visualization](/images/milestone_1/buy_eth_model.png)


在开始代码部分之前，我们首先来手动计算模型中用到的所有数学参数。为了简单起见，作者将使用Python来进行计算而不是Solidity，因为Solidity在数学计算上有很多细微之处需要考虑。也就是说，我们将会把所有的参数硬编码进池子合约里。这会让我们获得一个最小可用的产品。

本章中所有用到的python计算都在[unimath.py](https://github.com/Jeiwan/uniswapv3-code/blob/main/unimath.py)。

> 本章的完整代码可以参考[这个github分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1)
