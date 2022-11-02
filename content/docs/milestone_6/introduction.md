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

# NFT Positions

这是本书的一个锦上添花的部分。在这个 milestone 中，我们将学习如何扩展 Uniswap 合约，以及将它集成到第三方的协议中。能够实现这样的功能，正是由于我们把仅有关键函数的核心合约与外围合约分离开，让我们能够不需要在核心协议上添加功能就能集成到其他合约。

Uniswap V3 的一个额外特性是能够把流动性位置转换成 NFT。这是其中一个的例子：

![Uniswap V3 NFT example](/images/milestone_6/nft_example.png)

它展示了 token 的标识，池子费率，位置 ID，上下界 tick，token 地址，以及流动性所在的一段曲线。

> 你可以在[这个 Opensea 集合](https://opensea.io/collection/uniswap-v3-positions) 看到所有的 Uniswap V3 NFT positions。

在这个 milestone 中，我们将会添加使得流动性位置 NFT 代币化的功能。让我们开始吧！

> 你可以在[这个 Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_6) 找到本章完整代码.

> 本章对已有的合约做出了大量改动。 [在这里可以看到相比于上一章所有的代码变动](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_5...milestone_6)

> 如果你有任何问题，欢迎在[本章的 Github Discussion 中提问和交流](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-6-nft-positions)!