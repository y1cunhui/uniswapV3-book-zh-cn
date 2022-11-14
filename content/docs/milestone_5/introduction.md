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

# 费率和价格预言机

在这个 milestone 中，我们会在我们的实现中增加两个新的功能。它们的共同点是：都运行在我们已经搭建好的系统之上——这也是我们为什么直到这里才实现它们。然而，它们并不是同等重要的。

我们将要添加交易费率(swap fees)和一个价格预言机(price oracle)：
- 交易费率是我们实现的 DEX 中一个关键的机制。它能够使一切事情联合起来共同运作。交易费率能够激励 LP 提供流动性，没有流动性就无法进行交易。
- 一个价格预言机，这只是 DEX 的一个可选功能。一个 DEX 除了能够执行交易之外，也能够作为一个价格预言机——向其他服务提供 token 的价格。这实际上并不影响我们交易的执行，但是它对于其他链上应用来说是一个很有用的服务。

好，让我们来开始搭建吧！

> 你可以在[这个 Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_5) 找到本章完整代码.

> 本章对已有的合约做出了大量改动。 [在这里可以看到相比于上一章所有的代码变动](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_4...milestone_5)

> 如果你有任何问题，欢迎在[本章的 Github Discussion 中提问和交流](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-5-fees-and-price-oracle)!
