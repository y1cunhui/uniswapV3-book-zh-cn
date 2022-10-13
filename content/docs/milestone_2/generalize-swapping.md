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

# 通用swap

本节将会是这个milestone中最难的一个部分。在更新代码之前，我们首先需要知道Uniswap V3中的swap是如何工作的。

我们可以把一笔交易看作是满足一个订单：一个用户提交了一个订单，需要从池子中购买一定数量的某种token。池子会使用可用的流动性来将投入的token数量“转换”成输出的token数量。如果在当前价格区间中没有足够的流动性，它将会尝试在其他价格区间中寻找流动性（使用我们前一节实现的函数）。

现在，我们要实现`swap`函数内部的逻辑，但仍然保证交易可以在当前价格区间内完成——跨tick的交易将会在下一个milestone中实现。

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
```

在`swap`函数中，我们新增了两个参数：`zeroForOne` 和 `amountSpecified`。`zeroForOne` 是用来控制交易方向的 flag：当设置为true，是用 `token0` 兑换 `token1`；false则相反。例如，如果`token0` 是ETH，`token1` 是USDC，将 `zeroForOne` 设置为true意味着用 USDC 购买 ETH。`amountSpecified` 是用户希望卖出的token数量。

## 填满订单

由于在Uniswap V3中，流动性存储在不同的价格区间中，池子合约需要找到“填满当前订单”所需要的所有流动性。这个操作是通过沿着某个方向遍历所有初始化的tick来实现的。

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

`SwapState` 维护了当前swap的状态。`amoutSpecifiedRemaining` 跟踪了还需要从池子中获取的token数量：当这个数量为0时，这笔订单就被填满了。`amountCalculated` 是由合约计算出的输出数量。`sqrtPriceX96` 和 `tick` 是交易结束后的价格和`tick`。

`StepState` 维护了当前交易“一步”的状态。这个结构体跟踪“填满订单”过程中**一个循环**的状态。`sqrtPriceStartX96` 跟踪循环开始时的价格。`nextTick` 是能够为交易提供流动性的下一个已初始化的`tick`，`sqrtPriceNextX96` 是下一个`tick` 的价格。`amountIn` 和 `amountOut` 是当前循环中流动性能够提供的数量。

> 在我们实现跨tick的交易后（也即不发生在一个价格区间中的交易），关于循环方面会有更清晰的了解

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

在循环中，我们设置一个价格区间为这笔交易提供流动性的价格区间。这个区间是从 `state.sqrtPriceX96` 到 `step.sqrtPriceNextX96`，后者是下一个初始化的tick对应的价格（从上一章实现的`nextInitializedTickWithinOneWord` 中获取）。

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        step.sqrtPriceNextX96,
        liquidity,
        state.amountSpecifiedRemaining
    );
```

Next, we're calculating the amounts that can be provider by the current price range, and the new current price the swap
will result in.

```solidity
    state.amountSpecifiedRemaining -= step.amountIn;
    state.amountCalculated += step.amountOut;
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

Final steps in the loop is updating the SwapState. `step.amountIn` is the amount of tokens the price range can buy
from user; `step.amountOut` is the related number of the other token the pool can sell to user. `state.sqrtPriceX96` is
the current price that will be set after the swap (recall that trading changes current price).

## SwapMath Contract

Let's look closer at `SwapMath.computeSwapStep`.

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

This is the core logic of swapping. The function calculates swap amounts within one price range and respecting available
liquidity. It'll return: the new current price and input and output token amounts. Even though the input amount is provided
by user, we still calculate it to know how much of the user specified input amount was processed by one call to `computeSwapStep`.

```solidity
bool zeroForOne = sqrtPriceCurrentX96 >= sqrtPriceTargetX96;

sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
    sqrtPriceCurrentX96,
    liquidity,
    amountRemaining,
    zeroForOne
);
```

By checking the price, we can determine the direction of the swap. Knowing the direction, we can calculate the price after
swapping `amountRemaining` of tokens. We'll return to this function below.

After finding the new price, we can calculate input and output amounts of the swap using the function we already have (
the same functions we used to calculate token amounts from liquidity in the `mint` function):
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

That's it for `computeSwapStep`!

## Finding Price by Swap Amount

Let's now look at `Math.getNextSqrtPriceFromInput`–the function calculates a $\sqrt{P}$ given another $\sqrt{P}$,
liquidity, and input amount. It tells what the price will be after swapping the specified input amount of tokens, given
the current price and liquidity.

Good news is that we already know the formulas: recall how we calculated `price_next` in Python:
```python
# When amount_in is token0
price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))
# When amount_in is token1
price_next = sqrtp_cur + (amount_in * q96) // liq
```

We're going to implement this in Solidity:
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

The function handles swapping in both directions. Since calculations are different, we'll implement them in separate 
functions.

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
In this function, we're implementing two formulas. At the first `return`, it implements the same formula we implemented
in Python. This is the most precise formula, but it can overflow when multiplying `amountIn` by `sqrtPriceX96`. The
formula is (we discussed it in "Output Amount Calculation"):
$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

When it overflows, we use an alternative formula, which is less precise:
$$\sqrt{P_{target}} = \frac{L}{\Delta x + \frac{L}{\sqrt{P}}}$$

Which is simply the previous formula with the numerator and the denominator divided by $\sqrt{P}$ to get rid of the multiplication
in the numerator.

The other function has simpler math:
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

## Finishing the Swap

Now, let's return to the `swap` function and finish it.

By this moment, we have looped over next initialized ticks, filled `amountSpecified` specified by user, calculated input
and amount amounts, and found new price and tick. Since, in this milestone, we're implementing only swaps within one price
range, this is enough. We now need to update contract's state, send tokens to user, and get
tokens in exchange.


```solidity
if (state.tick != slot0_.tick) {
    (slot0.sqrtPriceX96, slot0.tick) = (state.sqrtPriceX96, state.tick);
}
```
First, we set new price and tick. Since this operation writes to contract's storage, we want to do it only if the new
tick is different, to optimize gas consumption.

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

Next, we calculate swap amounts based on swap direction and the amounts calculated during the swap loop.

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

Next, we're exchanging tokens with user, depending on swap direction. This piece is identical to what we had in Milestone 2,
only handling of the other swap direction was added.

That's it! Swapping is done!

## Testing

Test won't change significantly, we only need to pass `amountSpecified` and `zeroForOne` to `swap` function. Output amount
will change insignificantly though, because it's now calculated in Solidity.

We can now test swapping in the opposite direction! I'll leave this for you, as a homework (just be sure to choose a
small input amount so the whole swap can be handled by our single price range). Don't hesitate peeking at [my tests](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_2/test/UniswapV3Pool.t.sol)
if this feels difficult!