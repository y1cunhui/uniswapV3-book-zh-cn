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
首先，我们需要知道如何计算交易出入的数量。同样，我们在这小节中也会硬编码我们希望交易的 USDC 数额，这里我们选择 42，也即花费 42 USDC 购买 ETH。

在决定了我们希望投入的资金量之后，我们需要计算我们会获得多少 token。在 Uniswap V2 中，我们会使用当前池子的资产数量来计算，但是在 V3 中我们有 $L$ 和 $\sqrt{P}$，并且我们知道在交易过程中，$L$ 保持不变而只有 $\sqrt{P}$ 发生变化（当在同一区间内进行交易时，V3 的表现和 V2 一致）。我们还知道如下公式：

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

并且，在这里我们知道了$\Delta y$。它正是我们希望投入的 42 USDC。因此，我们可以根据公式得出投入的 42 USDC 会对价格造成多少影响：


$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

在 V3 中，我们得到了**我们交易导致的价格变动**（回忆一下，交易使得现价沿着曲线移动）。知道了目标价格(target price)，合约可以计算出投入 token 的数量和输出 token 的数量。

我们将数字代入上述公式：

$$\Delta \sqrt{P} = \frac{42 \enspace USDC}{1517882343751509868544} = 2192253463713690532467206957$$

把差价加到现在的 $\sqrt{P}$，我们就能得到目标价格：

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

$$\sqrt{P_{target}} = 5604469350942327889444743441197$$

> 在 Python 中进行相应计算:
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

知道了目标价格，我们就能计算出投入 token 的数量和获得 token 的数量：


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

使用上述公式，在知道价格变动和流动性数量的情况下，我们能求出我们购买了多少 ETH，也即 $\Delta x$。一个需要注意的点是： $\Delta \frac{1}{\sqrt{P}}$ 不等于
$\frac{1}{\Delta \sqrt{P}}$！前者才是 ETH 价格的变动，并且能够用如下公式计算：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

我们知道了公式里面的所有数值，接下来将其带入即可（可能会在显示上有些问题）：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$\Delta \frac{1}{\sqrt{P}} = -0.00000553186106731426$$

接下来计算 $\Delta x$:

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

即 0.008396714242162698 ETH，这与我们第一次算出来的数量非常接近！注意到这个结果是负数，因为我们是从池子中提出 ETH。

## 实现swap

交易在 `swap` 函数中实现：
```solidity
function swap(address recipient)
    public
    returns (int256 amount0, int256 amount1)
{
    ...
```

此时，它仅仅接受一个 recipient 参数，即提出 token 的接收者。

首先，我们需要计算出目标价格和对应 tick，以及 token 的数量。同样，我们将会在这里硬编码我们之前计算出来的值：

```solidity
...
int24 nextTick = 85184;
uint160 nextPrice = 5604469350942327889444743441197;

amount0 = -0.008396714242162444 ether;
amount1 = 42 ether;
...
```

接下来，我们需要更新现在的 tick 和对应的 `sqrtP`：

```solidity
...
(slot0.tick, slot0.sqrtPriceX96) = (nextTick, nextPrice);
...
```

然后，合约把对应的 token 发送给 recipient 并且让调用者将需要的 token 转移到本合约：

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

我们使用 callback 函数来将控制流转移到调用者，让它转入 token，之后我们需要通过检查确认 caller 转入了正确的数额。

最后，合约释放出一个 `swap` 事件，使得该笔交易能够被监听到。这个事件包含了所有有关这笔交易的信息：

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

这样就完成了。这个函数的功能仅仅是将一些 token 发送到了指定的接收地址，并且从调用者处接受一定数量的另一种 token。随着本书的进展，这个函数会变得越来越复杂。

## 测试交易

现在，我们来测试 `swap` 函数。在相同的测试文件中（即 `UniswapV3Pool.t.sol`），创建 `testSwapBuyEth` 函数并进行初始化设置。准备阶段的参数与 `testMintSuccess` 一致：


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

> 我们不会测试流动性是否正确添加到了池子里，因为之前已经有过针对此功能的测试样例了。

在测试中，我们需要 42 USDC：

```solidity
token1.mint(address(this), 42 ether);
```

交易之前，我们还需要实现 callback 函数，来确保能够将钱转给池子合约：

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
由于交易中的数额可以为正或负（从池子中拿走的数量），在 callback 中我们只发出数额为正的对应 token，也即我们希望卖出的 token。

现在我们可以调用 `swap` 了：

```solidity
(int256 amount0Delta, int256 amount1Delta) = pool.swap(address(this));
```

函数返回了在本次交易中涉及到的两种 token 数量，我们需要验证一下它们是否正确：

```solidity
assertEq(amount0Delta, -0.008396714242162444 ether, "invalid ETH out");
assertEq(amount1Delta, 42 ether, "invalid USDC in");
```
接下来，我们需要验证 token 的确从调用者（即本测试合约）处转出：

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

并且被发送到了池子合约中：
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

最后，我们验证池子的状态是否正确更新：

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

注意到，在这里交易并没有改变池子流动性——在后面的某个章节，我们会看到它将如何改变


## 练习

写一个测试样例，失败并报错 `InsufficientInputAmount`。要记得，这里还有一个隐藏的bug🙂
