---
title: "跨tick交易"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 跨 tick 交易

跨 tick 交易应该是 Uniswap V3 中最难的功能之一了。幸运地是，我们已经实现了我们需要完成跨 tick 交易的绝大部分内容。在实现它之前，让我们来简单看看它是怎么工作的：

## 跨 tick 交易如何工作

一个通常的 Uniswap V3 池子是一个有很多互相重叠的价格区间的池子。每个池子都会跟踪当前的 $\sqrt{P}$ 和 tick。当用户交易 token，他们会使得现价和 tick 向左或向右移动，移动方向取决于当前的交易方向。这样的移动是由于交易过程中某种 token 被添加进池子或者从池子中移除。

池子同样也会跟踪 $L$（代码中的 `liquidity` 变量），即**所有包含现价的价格区间提供的总流动性**。通常来说，如果价格移动幅度较大，现价会移出一些价格区间之外。这些时候，这种价格区间就会变为休眠，并且它们的流动性被从 $L$ 中减去。另一方面，如果现价进入了某个价格区间，这个价格区间就会被激活并且 $L$ 会增加。

让我们来分析这样一个场景：

![The dynamic of price ranges](/images/milestone_3/price_range_dynamics.png)

在图中有三个价格区间。最上面的一个是现在参与交易的区间，因为它包含现价。这个价格区间的流动性存储在池子合约中的 `liquidity` 变量中。

如果我们买走了最上面价格区间中的所有 ETH，价格会升高并且我们会移动到右侧的价格区间中，这个区间此时只包含 ETH 而不含 USDC。如果现在流动性已经足够满足我们的交易需求，我们可能就会停留在这个价格区间中；在这种情况下，`liquidity` 变量仅包含当前价格区间的所有流动性。如果我们继续购买 ETH 并且耗尽右边价格区间中的流动性，我们需要右边价格区间再有避难的某个区间。如果那里没有其他价格区间了，我们就不得不停下来，这笔交易将仅会部分成交。

如果我们从最上面的价格区间中买走所有的 USDC（并卖出 ETH），价格会下降并且我们会移动到左边的价格区间中——这个区间此时仅包含 USDC。如果我们耗尽这个区间，我们还需要再往左边的一个区间。

现价会在交易过程中移动。它从一个价格区间移动到另一个价格区间，但是它一定会在某个价格区间之内——否则，就没有流动性可以交易。

当然，价格区间之间是可以重叠的。所以在实际中，直接在两个价格区间之间转移不太可能发生。同样我们也不可能跳过一个没有流动性的部分——交易将会以部分成交而中止。同样值得注意的一点是，在价格区间重叠的部分，价格会移动得更缓慢。这是因为再这样的区域中供给量更高，所以需求的影响就会降低（在简介中我们说过，高需求低供应量会使得价格上升）。

我们现有的实现并不支持上述这样的灵活性——我们只支持了在单个活跃价格区间内部的交易。这也是我们本小节需要改进的地方。

## 更新 `computeSwapStep` 函数

在 `swap` 函数中，我们会沿着已初始化的 tick（有流动性的 tick）循环，直到用户需求的数量被满足。在每次循环中，我们会：
1. 使用 `tickBitmap.nextInitializedTickWithinOneWord` 来找到下一个已初始化的 tick；
2. 在现价和下一个已初始化的 tick 之间进行交易（使用 `SwapMath.computeSwapStep`）；
3. 总是假设当前流动性足够满足这笔交易（也即交易后的价格总在现价与下一个 tick 对应的价格之间）

但是如果上述的第三点不成立呢？我们在之前的测试中覆盖到了这一点：

```solidity
// test/UniswapV3Pool.t.sol
function testSwapBuyEthNotEnoughLiquidity() public {
    ...

    uint256 swapAmount = 5300 ether;

    ...

    vm.expectRevert(stdError.arithmeticError);
    pool.swap(address(this), false, swapAmount, extra);
}
```

当池子想要发送给我们超过它拥有数量的 ETH 时，会出发算数上溢/下溢的报错。这个错误是因为，在我们的实现中，我们期望总是有足够的流动性来满足任何交易：

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...

    sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
        sqrtPriceCurrentX96,
        liquidity,
        amountRemaining,
        zeroForOne
    );

    amountIn = ...
    amountOut = ...
}
```

为了改进上面函数，我们需要考虑以下几个场景：
1. 当现价和下一个 tick 之间的流动性足够填满 `amoutRemaining`；
2. 当这个区间不能填满 `amoutRemaining`。

在第一种情况中，交易会在当前区间中全部完成——这是我们已经实现的部分。在第二个场景中，我们会消耗掉当前区间所有流动性，并且**移动到下一个区间**（如果存在的话）。考虑到这点，我们来重新实现 `computeSwapStep`：

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...
    amountIn = zeroForOne
        ? Math.calcAmount0Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        )
        : Math.calcAmount1Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        );

    if (amountRemaining >= amountIn) sqrtPriceNextX96 = sqrtPriceTargetX96;
    else
        sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
            sqrtPriceCurrentX96,
            liquidity,
            amountRemaining,
            zeroForOne
        );

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
}
```

首先，我们计算 `amountIn`——当前区间可以满足的输入数量。如果它比 `amountRemaining` 要小，我们会说现在的区间不能满足整个交易，因此下一个 $\sqrt{P}$ 就会是当前区间的上界/下界（换句话说，我们使用了整个区间的流动性）。如果 `amountIn` 大于 `amountRemaining`，我们计算 `sqrtPriceNextX96`——一个仍然在现在区间内的价格。

最后，在找到下一个价格之后，我们在这个区间中重新计算 `amountIn` 并计算 `amountOut`。

希望上面没有让你感到迷惑！

## 更新 `swap` 函数

现在，在 `swap` 函数中，我们会处理我们在前一部分中提到的场景：当价格移动到了当前区间的边界处。此时，我们希望使得我们离开的当前区间休眠，并激活下一个区间。并且我们会开始下一个循环并且寻找下一个有流动性的 tick。

我们会在循环的尾部加这些：

```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(step.nextTick);

    if (zeroForOne) liquidityDelta = -liquidityDelta;

    state.liquidity = LiquidityMath.addLiquidity(
        state.liquidity,
        liquidityDelta
    );

    if (state.liquidity == 0) revert NotEnoughLiquidity();

    state.tick = zeroForOne ? step.nextTick - 1 : step.nextTick;
} else {
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

第二个分支是我们之前实现的——处理了交易仍然停留在当前区间的情况。所以我们主要关注第一个分支。

`state.sqrtPriceX96` 是新的现价，即在上一个交易过后会被设置的价格；`step.sqrtNextX96` 是下一个已初始化的 tick 对应的价格。如果它们相等，说明我们达到了这个区间的边界。正如之前所说，此时我们需要更新 $L$（添加或移除流动性）并且使用这个边界 tick 作为现在的 tick，继续这笔交易。

通常来说，穿过一个 tick 是指从左到右穿过。因此，穿过一个下界 tick 或增加流动性，穿过一个上界 tick 会减少流动性。然而如果 `zeroForOne` 被设置为 true，我们会把符号反过来：当价格下降时，上界 tick 会增加流动性，下界 tick 会减少流动性。

当更新 `state.tick` 时，如果价格是下降的（`zeroForOne` 设置为 true），我们需要将 tick 减一来走到下一个区间；而当价格上升时（`zeroForOne` 为 false），根据 `TickBitmap.nextInitializedTickWithinOneWord`，已经走到了下一个区间了。

（译者注：开闭区间的问题，一个方向需要 +1）

另一个重要的改动是，当我们需要在跨过 tick 时更新流动性。全局的更新是在循环之后：

```solidity
if (liquidity_ != state.liquidity) liquidity = state.liquidity;
```

在区间内，我们在进入/离开区间时多次更新 `state.liquidity`。交易后，我们需要更新全局的 $L$ 来反应现价可用的流动性，同时避免多次写合约状态而消耗 gas。

## 流动性跟踪以及 tick 的跨域

现在让我们来更新 `Tick` 库。

首先要更改的是 `Tick.Info` 结构体：我们现在需要两个变量来跟踪 tick 的流动性：

```solidity
struct Info {
    bool initialized;
    // total liquidity at tick
    uint128 liquidityGross;
    // amount of liqudiity added or subtracted when tick is crossed
    int128 liquidityNet;
}
```

`liquidityGross` 跟踪一个tick拥有的绝对流动性数量。它用来跟踪一个 tick 是否还可用。`liquidityNet`，是一个有符号整数，用来跟踪当跨越 tick 时添加/移除的流动性数量。

`liquidityNet` 在 `update` 函数中设置:
```solidity
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int128 liquidityDelta,
    bool upper
) internal returns (bool flipped) {
    ...

    tickInfo.liquidityNet = upper
        ? int128(int256(tickInfo.liquidityNet) - liquidityDelta)
        : int128(int256(tickInfo.liquidityNet) + liquidityDelta);
}
```

上面我们提到的 `cross` 函数的功能也就是返回 `liquidityNet`（在后面的 milestone 我们引入更多功能时，函数会变得更复杂）：

```solidity
function cross(mapping(int24 => Tick.Info) storage self, int24 tick)
    internal
    view
    returns (int128 liquidityDelta)
{
    Tick.Info storage info = self[tick];
    liquidityDelta = info.liquidityNet;
}
```

## 测试

现在我们来在各种不同的流动性场景下测试我们的实现是否能正确处理。

### 单个价格区间

![Swap within price range](/images/milestone_3/swap_within_price_range.png)

这就是我们之前所用的场景。当我们更新代码后，我们仍然需要确保之前的功能仍然正常工作。

> 为了文章的简洁，这里仅仅展示代码的关键部分。完整的代码可以参考[这里](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_3/test/UniswapV3Pool.Swaps.t.sol)

- 购买 ETH:
    ```solidity
    function testBuyETHOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008396874645169943 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5604415652688968742392013927525, // 5003.8180249710795
                tick: 85183,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```
- 购买 USDC:
    ```solidity
    function testBuyUSDCOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.807123823853842027 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5598737223630966236662554421688, // 4993.683362269102
                tick: 85163,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```

在两个场景中我们都只购买了小额的 ETH 或者 USDC——金额需要足够小来保证不会离开当前价格区间。交易完成时需要注意的关键点：
1. `sqrtPriceX96` 略高于或者略低于之前的价格，并且仍然在价格区间内；
2. `curerntLiquidity` 保持不变。

## 多个相同的重叠价格区间 


### Multiple Identical and Overlapping Price Ranges

- 购买 ETH:
    ```solidity
    function testBuyETHTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008398516982770993 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5603319704133145322707074461607, // 5001.861214026131
                tick: 85179,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

- 购买 USDC:
    ```solidity
    function testBuyUSDCTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.827918929906650442 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5600479946976371527693873969480, // 4996.792621611429
                tick: 85169,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

这个场景与之前的类似，但是在这里我们创建了两个完全相同的价格区间。由于这两个区间完全重叠，实际上就相当于一个流动性更高的单个区间。因此，相比于前一个场景，这里的价格移动更缓慢。并且，我们会比之前得到多一点的 token，这是因为更深的流动性。

### 连续的价格区间

![Swap over consecutive price ranges](/images/milestone_3/swap_consecutive_price_ranges.png)

- 购买 ETH:
    ```solidity
    function testBuyETHConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5500, 6250, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.820694594787485635 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6190476002219365604851182401841, // 6105.045728033458
                tick: 87173,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```
- 购买 USDC:
    ```solidity
    function testBuyUSDCConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4545, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9103.264925902176327184 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5069962753257045266417033265661, // 4094.9666586581643
                tick: 83179,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

在这个场景中，我们交易数额较大，使得价格移出了当前区间。此时第二个价格区间被激活，并且提供了足够的流动性来完成这笔交易。在两个测试中，我们都可以看到最后的价格超出了初始的价格区间，并且初始价格区间被停用。（`currentLiquidity` 等于第二个区间的流动性）。

### 部分重叠的价格区间

![Swap over partially overlapping price ranges](/images/milestone_3/swap_partially_overlapping_price_ranges.png)

- 购买 ETH:
    ```solidity
    function testBuyETHPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5001, 6250, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.864220641170389178 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6165345094827913637987008642386, // 6055.578153852725
                tick: 87091,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

- 购买 USDC:
    ```solidity
    function testBuyUSDCPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4999, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9321.077831210790476918 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5090915820491052794734777344590, // 4128.883835866256
                tick: 83261,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

这是上一个场景的一个变种，此时两个价格区间部分重叠。在两个区间重叠的区域，流动性更深，价格移动更慢，相当于在重叠的部分提供了更多的流动性。

另外也可以注意到，在两个方向的交易中，我们都比“连续价格区间”场景下得到了更多的 token——这也是由重叠区间更深的流动性导致的。