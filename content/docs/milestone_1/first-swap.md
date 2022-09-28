---
title: "第一笔交易"
weight: 4
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}


# 第一笔交易

现在我们已经有了流动性，我们可以进行我们的第一笔交易了！

## 计算交易数量
首先，我们需要知道如何计算交易出入的数量。同样，我们在这小节中也会硬编码我们希望交易的USDC数额，这里我们选择42，也即花费42USDC购买ETH

在决定了我们希望投入的资金量之后，我们需要计算我们会获得多少token。在Uniswap V2中，我们会使用现在池子的资产数量来计算，但是在V3中我们有$L$和$\sqrt{P}$，并且我们知道在交易过程中，$L$保持不变而只有$\sqrt{P}$发生变化（当在同一区间内进行交易时，V3的表现和V2一致）。我们还知道如下公式：

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

并且，在这里我们知道了$\Delta y$！它正是我们希望投入的42USDC。因此，我们可以根据公式得出投入的 42USDC 会对价格造成多少影响：


$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

在V3中，我们选择**我们希望交易导致的价格变动**（回忆一下，交易使得现价沿着曲线移动）。知道了目标价格(target price)，合约可以计算出投入token的数量和输出token的数量。

我们将数字代入上述公式：


$$\Delta \sqrt{P} = \frac{42 \enspace USDC}{1517882343751509868544} = 2192253463713690532467206957$$

把差价加到现在的$\sqrt{P}$，我们就能得到目标价格：

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

$$\sqrt{P_{target}} = 5604469350942327889444743441197$$

> 在Python中进行相应计算:
> ```python
> amount_in = 42 * eth
> price_diff = (amount_in * q96) // liq
> price_next = sqrtp_cur + price_diff
> print("New price:", (price_next / q96) ** 2)
> print("New sqrtP:", price_next)
> print("New tick:", price_to_tick((price_next / q96) ** 2))
> # New price: 5003.913912782393
> # New sqrtP: 5604469350942327889444743441197
> # New tick: 85184
> ```

知道了目标价格，我们就能计算出投入token的数量和获得token的数量：


$$ x = \frac{L(\sqrt{p_b}-\sqrt{p_a})}{\sqrt{p_b}\sqrt{p_a}}$$
$$ y = L(\sqrt{p_b}-\sqrt{p_a}) $$

> 用Python:
> ```python
> amount_in = calc_amount1(liq, price_next, sqrtp_cur)
> amount_out = calc_amount0(liq, price_next, sqrtp_cur)
> 
> print("USDC in:", amount_in / eth)
> print("ETH out:", amount_out / eth)
> # USDC in: 42.0
> # ETH out: 0.008396714242162444
> ```

我们使用另一个公式验证一下：

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

使用上述公式，在知道价格变动和流动性数量的情况下，我们能求出我们购买了多少ETH，也即$\Delta x$。一个需要注意的点是： $\Delta \frac{1}{\sqrt{P}}$ 不等于
$\frac{1}{\Delta \sqrt{P}}$！前者才是ETH价格的变动，并且能够用如下公式计算：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

我们知道了公式里面的所有数值，接下来将其带入即可（可能会在屏幕显示上有些问题）：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$\Delta \frac{1}{\sqrt{P}} = -0.00000553186106731426$$

接下来计算 $\Delta x$:

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

即 0.008396714242162698 ETH，这与我们第一次算出来的数量非常接近！注意到这个结果是负数，因为我们是从池子中移出ETH。

## Implementing a Swap

Swapping is implemented in `swap` function:
```solidity
function swap(address recipient)
    public
    returns (int256 amount0, int256 amount1)
{
    ...
```
At this moment, it only takes a recipient, who is a receiver of tokens.

First, we need to find the target price and tick, as well as calculate the token amounts. Again, we'll simply hard code
the values we calculated earlier to keep things as simple as possible:
```solidity
...
int24 nextTick = 85184;
uint160 nextPrice = 5604469350942327889444743441197;

amount0 = -0.008396714242162444 ether;
amount1 = 42 ether;
...
```

Next, we need to update the current tick and `sqrtP` since trading affects the current price:
```solidity
...
(slot0.tick, slot0.sqrtPriceX96) = (nextTick, nextPrice);
...
```

Next, the contract sends tokens to the recipient and lets the caller transfer the input amount into the contract:
```solidity
...
IERC20(token0).transfer(recipient, uint256(-amount0));

uint256 balance1Before = balance1();
IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
    amount0,
    amount1
);
if (balance1Before + uint256(amount1) < balance1())
    revert InsufficientInputAmount();
...
```

Again, we're using a callback to pass the control to the caller and let it transfer the tokens. After that, we're checking
that pool's balance is correct and includes the input amount.

Finally, the contract emits a `Swap` event to make the swap discoverable. The event includes all the information about
the swap:
```solidity
...
emit Swap(
    msg.sender,
    recipient,
    amount0,
    amount1,
    slot0.sqrtPriceX96,
    liquidity,
    slot0.tick
);
```

And that's it! The function simply sends some amount of tokens to the specified recipient address and expects a certain
number of the other token in exchange. Throughout this book, the function will get much more complicated.

## Testing Swapping

Now, we can test the swap function. In the same test file, create `testSwapBuyEth` function and set up the test case. This
test case uses the same parameters as `testMintSuccess`:
```solidity
function testSwapBuyEth() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
    (uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

    ...
```

Next steps will be different, however.

> We're not going to test that liquidity has been correctly added to the pool since we tested this functionality in the
other test cases.

To make the test swap, we need 42 USDC:
```solidity
token1.mint(address(this), 42 ether);
```

Before making the swap, we need to ensure we can transfer tokens to the pool contract when it requests them:
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```
Since amounts during a swap can be positive (the amount that's sent to the pool) and negative (the amount that's taken
from the pool), in the callback, we only want to send the positive amount, i.e. the amount we're trading in.

Now, we can call `swap`:
```solidity
(int256 amount0Delta, int256 amount1Delta) = pool.swap(address(this));
```

The function returns token amounts used in the swap, and we can check them right away:
```solidity
assertEq(amount0Delta, -0.008396714242162444 ether, "invalid ETH out");
assertEq(amount1Delta, 42 ether, "invalid USDC in");
```

Then, we need to ensure that tokens were actually transferred from the caller:
```solidity
assertEq(
    token0.balanceOf(address(this)),
    uint256(userBalance0Before - amount0Delta),
    "invalid user ETH balance"
);
assertEq(
    token1.balanceOf(address(this)),
    0,
    "invalid user USDC balance"
);
```

And sent to the pool contract:
```solidity
assertEq(
    token0.balanceOf(address(pool)),
    uint256(int256(poolBalance0) + amount0Delta),
    "invalid pool ETH balance"
);
assertEq(
    token1.balanceOf(address(pool)),
    uint256(int256(poolBalance1) + amount1Delta),
    "invalid pool USDC balance"
);
```

Finally, we're checking that the pool state was updated correctly:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5604469350942327889444743441197,
    "invalid current sqrtP"
);
assertEq(tick, 85184, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

Notice that swapping doesn't change the current liquidity–in a later chapter, we'll see when it does change it.

## Homework

Write a test that fails with `InsufficientInputAmount` error. Keep in mind that there's a hidden bug 🙂