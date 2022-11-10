---
title: "Solidity中的数学运算"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Solidity中的数学运算

由于 Solidity 不支持浮点数，在其中的运算会有些复杂。Solidity 拥有整数(integer)和无符号整数(unsigned integer)类型，这并不足够让我们实现复杂的数学运算。

另一个困难之处在于 gas 消耗：一个算法越复杂，它消耗的 gas 就越多。因此，如果我们需要比较高级的数学运算（例如`exp`, `ln`, `sqrt`），我们会希望它们尽可能节省 gas。

还有一个很大的问题是溢出。当进行 `uint256` 类型的乘法时，有溢出的风险：结果的数据可能会超出256位。

上述这些困难让我们不得不选择使用那些实现了高级数学运算并进行了gas优化的第三方库。如果库里面没有我们需要的算法，我们就需要自己来实现，这将会很困难。

## 重用数学库

在我们的 Uniswap V3 实现中，我们会使用两个第三方数学库：
1. [PRBMath](https://github.com/paulrberg/prb-math)，一个包含了复杂的定点数运算的库。我们会使用其中的` mulDiv` 函数来处理乘除法过程中可能的溢出
2. [TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol)，来自原始的 Uniswap V3 仓库。这个合约实现了两个函数，`getSqrtRatioAtTick` 和 `getTickAtSqrtRatio`，功能是在 tick 和 $\sqrt{P}$ 之间相互转换。

我们先关注第二个库。在我们的合约中，我们需要将 tick 转换成对应的 $\sqrt{P}$，或者反过来。对应的公式为：

$$\sqrt{P(i)} = \sqrt{1.0001^i} = 1.0001^{\frac{i}{2}}$$

$$i = log_{\sqrt{1.0001}}\sqrt{P(i)}$$

这些是非常复杂的数学运算（至少在 Solidity 中是这样）并且它们需要很高的精度，因为我们不希望取整的问题干扰我们的价格计算。为了能实现更高的精度和 gas 优化，我们需要采用特定的实现。

如果你看一下 [getSqrtRatioAtTick](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L23-L54) 和 [getTickAtSqrtRatio](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L61-L204) 的代码，你会发现它们非常复杂：其中有大量的 magic number（像 `0xfffcb933bd6fad37aa2d162d1a594001` 这样），乘法以及位运算。在当前阶段，我们不会试图分析这些代码或者尝试重新实现它们，因为这会是另一个不同而且很复杂的主题。我们在这里仅仅使用它们。在后续的章节中，我们可能会深入探讨这部分代码。
