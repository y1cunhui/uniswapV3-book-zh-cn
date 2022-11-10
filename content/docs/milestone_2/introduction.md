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

# 第二笔交易

OK，现在才是真正的开始。到目前为止，我们的实现看起来过于静态。我们手动计算了所有参数，硬编码了各种数量，来让学习曲线不那么陡峭；现在我们准备要让它真正地自动化工作了。我们将会实现第二笔交易，这次的交易是相反的方向：卖出 ETH 来获得 USDC。为了达到这个目的，我们需要大幅度改进我们目前的合约：
1. 我们需要在 Solidity 中实现数学运算。但是，由于 Solidity 仅支持整数除法，在 Solidity 中实现数学运算会比较困难。我们将使用第三方库来完成这部分；
2. 我们需要让用户能够选择交易的方向，并且池子合约需要支持双向的交易。我们将会改进合约，离跨价格区间的交易更进一步，而我们将在下一个 milestone 真正实现它；
3. 最后，我们需要更新我们的 UI 来实现双向的交易以及获取金额的计算。这需要我们实现另一个合约，报价合约(Quoter)。

在本章节的最后，我们将会获得一个几乎和真正 DEX 类似的 app！

让我们开始吧

> 你可以在[这个 Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_2) 找到本章完整代码.

> 本章对已有的合约做出了大量改动。 [在这里可以看到相比于上一章所有的代码变动](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_1...milestone_2)

> 如果你有任何问题，欢迎在[本章的 Github Discussion 中提问和交流](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-2-second-swap)!



