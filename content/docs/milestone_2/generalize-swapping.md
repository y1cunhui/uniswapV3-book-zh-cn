---
title: "通用swap"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 通用 swap

本节将会是这个 milestone 中最难的一个部分。在更新代码之前，我们首先需要知道 Uniswap V3 中的 swap 是如何工作的。

我们可以把一笔交易看作是满足一个订单：一个用户提交了一个订单，需要从池子中购买一定数量的某种 token。池子会使用可用的流动性来将投入的 token 数量“转换”成输出的 token 数量。如果在当前价格区间中没有足够的流动性，它将会尝试在其他价格区间中寻找流动性（使用我们前一节实现的函数）。

现在，我们要实现 `swap` 函数内部的逻辑，但仍然保证交易可以在当前价格区间内完成——跨 tick 的交易将会在下一个 milestone 中实现。

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
```

在 `swap` 函数中，我们新增了两个参数：`zeroForOne` 和 `amountSpecified`。`zeroForOne` 是用来控制交易方向的 flag：当设置为 true，是用 `token0` 兑换 `token1`；false 则相反。例如，如果 `token0` 是ETH，`token1` 是USDC，将 `zeroForOne` 设置为 true 意味着用 USDC 购买 ETH。`amountSpecified` 是用户希望卖出的 token 数量。

## 填满订单

由于在 Uniswap V3 中，流动性存储在不同的价格区间中，池子合约需要找到“填满当前订单”所需要的所有流动性。这个操作是通过沿着某个方向遍历所有初始化的 tick 来实现的。

在继续之前，我们需要定义两个新的结构体：

```solidity
struct SwapState {
    uint256 amountSpecifiedRemaining;
    uint256 amountCalculated;
    uint160 sqrtPriceX96;
    int24 tick;
}

struct StepState {
    uint160 sqrtPriceStartX96;
    int24 nextTick;
    uint160 sqrtPriceNextX96;
    uint256 amountIn;
    uint256 amountOut;
}
```

`SwapState` 维护了当前 swap 的状态。`amoutSpecifiedRemaining` 跟踪了还需要从池子中获取的 token 数量：当这个数量为 0 时，这笔订单就被填满了。`amountCalculated` 是由合约计算出的输出数量。`sqrtPriceX96` 和 `tick` 是交易结束后的价格和 `tick`。

`StepState` 维护了当前交易“一步”的状态。这个结构体跟踪“填满订单”过程中**一个循环**的状态。`sqrtPriceStartX96` 跟踪循环开始时的价格。`nextTick` 是能够为交易提供流动性的下一个已初始化的`tick`，`sqrtPriceNextX96` 是下一个 `tick` 的价格。`amountIn` 和 `amountOut` 是当前循环中流动性能够提供的数量。

> 在我们实现跨 tick 的交易后（也即不发生在一个价格区间中的交易），关于循环方面会有更清晰的了解。

```solidity
// src/UniswapV3Pool.sol

function swap(...) {
    Slot0 memory slot0_ = slot0;

    SwapState memory state = SwapState({
        amountSpecifiedRemaining: amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0_.sqrtPriceX96,
        tick: slot0_.tick
    });
    ...
```

在填满一个订单之前，我们首先初始化 `SwapState` 的实例。我们将会循环直到 `amoutSpecified` 变成0，也即池子拥有足够的流动性来买用户的 `amountSpecified` 数量的token。

```solidity
...
while (state.amountSpecifiedRemaining > 0) {
    StepState memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        1,
        zeroForOne
    );

    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.nextTick);
```

在循环中，我们设置一个价格区间为这笔交易提供流动性的价格区间。这个区间是从 `state.sqrtPriceX96` 到 `step.sqrtPriceNextX96`，后者是下一个初始化的 tick 对应的价格（从上一章实现的 `nextInitializedTickWithinOneWord` 中获取）。

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        step.sqrtPriceNextX96,
        liquidity,
        state.amountSpecifiedRemaining
    );
```

接下来，我们计算当前价格区间能够提供的流动性的数量，以及交易达到的目标价格。

```solidity
    state.amountSpecifiedRemaining -= step.amountIn;
    state.amountCalculated += step.amountOut;
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

循环中的最后一步就是更新SwapState。`step.amountIn` 是这个价格区间可以从用户手中买走的token数量了；`step.amountOut` 是相应的池子卖给用户的数量。`state.sqrtPriceX96` 是交易结束后的现价（因为交易会改变价格）。

## SwapMath 合约

接下来，让我们更深入研究一下 `SwapMath.computeSwapStep`：

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(
    uint160 sqrtPriceCurrentX96,
    uint160 sqrtPriceTargetX96,
    uint128 liquidity,
    uint256 amountRemaining
)
    internal
    pure
    returns (
        uint160 sqrtPriceNextX96,
        uint256 amountIn,
        uint256 amountOut
    )
{
    ...
```

这是整个swap的核心逻辑所在。这个函数计算了一个价格区间内部的交易数量以及对应的流动性。它的返回值是：新的现价、输入 token 数量、输出 token 数量。尽管输入 token 数量是由用户提供的，我们仍然需要进行计算在对于 `computeSwapStep` 的一次调用中可以处理多少用户提供的 token。

```solidity
bool zeroForOne = sqrtPriceCurrentX96 >= sqrtPriceTargetX96;

sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
    sqrtPriceCurrentX96,
    liquidity,
    amountRemaining,
    zeroForOne
);
```
通过检查价格大小我们来确认交易的方向。知道交易方向后，我们就可以计算交易 `amountRemaining` 数量 token 之后的价格。在下面我们还会回过头来看这个函数。

找到新的价格后，我们根据之前已有的函数能够计算出输入和输出的数量（与 `mint` 里面用到的，根据流动性计算 token 数量的函数相同）：

```solidity
amountIn = Math.calcAmount0Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
amountOut = Math.calcAmount1Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
```

And swap the amounts if the direction is opposite:
```solidity
if (!zeroForOne) {
    (amountIn, amountOut) = (amountOut, amountIn);
}
```

这就是 `computeSwapStep` 的全部！

## 通过交易数量获取价格

接下来我们来看 `Math.getNextSqrtPriceFromInput` 函数——这个函数根据现在的 $\sqrt{P}$、流动性、和输入数量，计算出交易后新的 $\sqrt{P}$。

一个好消息是我们已经知道了相关的公式。回忆一下，我们之前在 Python 中计算 `price_next`

```python
# When amount_in is token0
price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))
# When amount_in is token1
price_next = sqrtp_cur + (amount_in * q96) // liq
```

我们会在 Solidity 中实现上述功能：
```solidity
// src/lib/Math.sol
function getNextSqrtPriceFromInput(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtPriceNextX96) {
    sqrtPriceNextX96 = zeroForOne
        ? getNextSqrtPriceFromAmount0RoundingUp(
            sqrtPriceX96,
            liquidity,
            amountIn
        )
        : getNextSqrtPriceFromAmount1RoundingDown(
            sqrtPriceX96,
            liquidity,
            amountIn
        );
}
```
这个函数仅仅是分别处理了两个方向的功能。我们会分别在两个不同的函数中进行实现：

```solidity
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    uint256 numerator = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 product = amountIn * sqrtPriceX96;

    if (product / amountIn == sqrtPriceX96) {
        uint256 denominator = numerator + product;
        if (denominator >= numerator) {
            return
                uint160(
                    mulDivRoundingUp(numerator, sqrtPriceX96, denominator)
                );
        }
    }

    return
        uint160(
            divRoundingUp(numerator, (numerator / sqrtPriceX96) + amountIn)
        );
}
```

在这个函数中，我们实现了两个公式。在第一个 `return` 那里，实现了我们 Python 中提到的公式。这是最精确的公式，但是它可能会在 `amountIn` 与 `sqrtPriceX96` 相乘时产生溢出。公式是：

$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

当它可能产生溢出时，我们使用另一个替代的公式，精确度会更低但是不会溢出：

$$\sqrt{P_{target}} = \frac{L}{\Delta x + \frac{L}{\sqrt{P}}}$$

其实也仅仅是把第一个公式上下同时除以 $\sqrt{P}$ 得到的。

另一个函数的实现会简单一些：
```solidity
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    return
        sqrtPriceX96 +
        uint160((amountIn << FixedPoint96.RESOLUTION) / liquidity);
}
```

## 完成 swap

现在，让我们回到 `swap` 函数并且完成它。

到目前为止，我们已经能够沿着下一个初始化过的tick进行循环、填满用户指定的 `amoutSpecified`、计算输入和输出数量，并且找到新的价格和 tick。由于在本章节中我们只实现在一个价格区间内的交易，这些功能就已经足够了。我们现在只需要去更新合约状态、将 token 发送给用户，并从用户处获得 token。

```solidity
if (state.tick != slot0_.tick) {
    (slot0.sqrtPriceX96, slot0.tick) = (state.sqrtPriceX96, state.tick);
}
```

首先，我们设置新的价格和 tick。由于这个操作需要对合约的存储进行写操作，我们仅仅会在新的 tick 不同的时候进行更新，来节省 gas。

```solidity
(amount0, amount1) = zeroForOne
    ? (
        int256(amountSpecified - state.amountSpecifiedRemaining),
        -int256(state.amountCalculated)
    )
    : (
        -int256(state.amountCalculated),
        int256(amountSpecified - state.amountSpecifiedRemaining)
    );
```
接下来，我们根据交易的方向来获得循环中计算出的对应数量。


```solidity
if (zeroForOne) {
    IERC20(token1).transfer(recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance0Before + uint256(amount0) > balance0())
        revert InsufficientInputAmount();
} else {
    IERC20(token0).transfer(recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance1Before + uint256(amount1) > balance1())
        revert InsufficientInputAmount();
}
```

接下来，我们根据交易方向与用户交换 token。这个部分和我们在 milestone 1 中实现的部分一样，除了要考虑交易方向之外。

现在 Swap 已经完成了！

## 测试

测试的改变并不大，我们仅仅需要把 `amoutSpecified` 和 `zeroForOne` 传参给 `swap` 函数。输出的数量会略微有不同，因为这里是用 Solidity 计算的。

我们现在也可以测试相反方向的交易了！此测试留作作业给读者完成（记得选择一个较小的金额，来确保交易发生在同一个价格区间）。如果遇到困难，可以参考[作者的测试](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_2/test/UniswapV3Pool.t.sol)。
