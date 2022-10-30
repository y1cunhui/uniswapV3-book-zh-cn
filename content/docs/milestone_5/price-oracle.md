---
title: "价格预言机"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 价格预言机

我们要在我们的 DEX 中添加的最后一个机制就是*价格预言机*。即使它不是一个 DEX 必备的部分（有很多 DEX 没有价格预言机），它仍然是 Uniswap 的一个很重要的特性，并且也非常有趣。

## 什么是价格预言机？

价格预言机(Price Oracle)是向区块链提供资产价格的一种机制。由于区块链是一个独立的生态系统，我们没有很直接的方法能够读取到外部数据，例如从中心化交易所(CEX)中通过 API 获取价格。另一个非常重要的问题是数据的有效性和权威性：当你从一个交易所获取数据时，你怎么能确定它是真实的？你必须信任数据来源。但是网络并不总是安全的，并且价格有时候会被操纵，DNS 记录可能被污染，API 服务器可能会宕机等等。所有这些的困难都需要被考虑到，因为我们需要可靠而且正确的链上价格。

上述问题最初的解决方法之一是 [Chainlink](https://chain.link/)。Chainlink 运行了一个去中心化的预言机网络，它通过 API 从中心化交易所那里获取资产价格，取平均数，然后用防篡改的方式向链上提供数据。简单来说，Chainlink 有一系列的合约拥有一个状态变量，即资产价格，任何人可以读到这个变量（其他合约或者用户），但它仅能被预言机修改。

这只是解决价格预言机的一种方式。还有其他的方式。

如果我们已经有了链上原生的交易所，那我们为什么还需要从外界获取价格呢？这正是 Uniswap 价格预言机工作的原理。多亏了高流动性以及套利，Uniswap 中的资产价格与中心化交易所中的价格非常接近。因此，除了使用中心化交易所作为可信的资产价格来源，我们还可以使用 Uniswap，并且我们不再需要考虑关于把数据传输到链上的问题（我们也不需要再信任数据提供方）。

## Uniswap 价格预言机如何工作

Uniswap 仅仅记录了所有之前的交易价格，就是这样。但是 Uniswap 并没有直接使用实际价格，而是跟踪 *累积价格*，也即这个池子合约历史中每秒的价格之和。

$$a_{i} = \sum_{i=1}^t p_{i}$$

这个方法让我们能够找到两个时间点($t_1$ 和 $t_2$)之间的*时间加权平均价格(TWAP)*，只需要将这两个时间点的累积价格($a_{t_1}$ 和 $a_{t_2}$)相减，除以他们之间相隔的秒数即可：

$$p_{t_1,t_2} = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1}$$

Uniswap V2 中是这样工作的。在 V3 中，略有一些不同。累积的价格通过 tick 来计算的（也即价格的$log_{1.0001}$）：

$$a_{i} = \sum_{i=1}^t log_{1.0001}P(i)$$

而且不再是直接平均价格，而是采用*几何平均数*：

$$ P_{t_1,t_2} = \left( \prod_{i=t_1}^{t_2} P_i \right) ^ \frac{1}{t_2-t_1} $$

为了求出两个之间点之间的时间加权几何平均价格，我们用这两个时间点的累积价格详见，除以两点之间的秒数差，然后计算$1.0001^{x}$：

$$ log_{1.0001}{(P_{t_1,t_2})} = \frac{\sum_{i=t_1}^{t_2} log_{1.0001}(P_i)}{t_2-t_1}$$
$$ = \frac{a_{t_2} - a_{t_1}}{t_2-t_1}$$

$$P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2-t_1}}$$

Uniswap V2 并不存储历史累积价格，这就需要第三方的链上数据服务来自行存储并计算平均价格。而 Uniswap V3，允许存储最多 65535 个历史累积价格，使得获取历史时间加权几何平均价格更容易。

## 减轻价格操控

另一个重要的点是价格操控，以及如何在 Uniswap 中减小它的影响。

理论上，我们可以操纵池子的价格来获得利益：例如，买走大量的 token 来提高价格，然后从使用 Uniswap 价格预言机的第三方服务中获取利润，最后卖出 token 回到原价。为了减少这样的攻击，Uniswap 跟踪**区块结束时**的价格，在一个区块的最后一笔交易*之后*。这消除了在同一个区块内部的价格操纵的可能性。

实际上，Uniswap 预言机中的价格在每个区块的头部更新，并且每个价格在区块的第一笔交易之后计算。

## 价格预言机的实现

现在让我们开始代码部分。

### 观测和基数

我们首先创建 `Oracle` 库，以及 `Observation` 结构体：

```solidity
// src/lib/Oracle.sol
library Oracle {
    struct Observation {
        uint32 timestamp;
        int56 tickCumulative;
        bool initialized;
    }
    ...
}
```

*一个观测(observation)* 是存储一个记录价格的 slot。它存储一个价格，记录这个价格时的时间戳，以及一个 `initialized` 标志位，当这个观测被激活时设置为 true（并不是所有的观测都默认激活）。一个池子合约能够存储至多 65535 个观测：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    using Oracle for Oracle.Observation[65535];
    ...
    Oracle.Observation[65535] public observations;
}
```

然而，由于存储这么多 `Observation` 的实例会需要大量的 gas（总有人要为大量合约存储的写操作付款），一个池子默认只存储一个观测，每次新价格记录时都会把它覆盖掉。激活的观测数量，也即观测的*基数(cardinality)*，可以由任何愿意付款的人在任何时间增加。为了管理基数，我们需要一些额外的状态变量：

```solidity
    ...
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
        // Most recent observation index
        uint16 observationIndex;
        // Maximum number of observations
        uint16 observationCardinality;
        // Next maximum number of observations
        uint16 observationCardinalityNext;
    }
    ...
```

- `observationIndex` 记录最新的观测的编号；
- `observationCardinality` 记录活跃的观测数量；
- `observationCardinalityNext` 记录观测数组能够扩展到的下一个基数的大小。

观测存储在一个定长的数组里，当一个新的观测被存储并且 `observationCardinalityNext` 超过 `observationCardinality` 的时候就会扩展。如果一个数组不能被扩展（下一个基数与现在的基数相同），旧的观测就会被覆盖，例如一个观测存储在下标 0，下一个就存储在下标 1，以此类推。

当池子创建的时候，`observationCardinality` 和 `observationCardinalityNext` 都设置为1：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function initialize(uint160 sqrtPriceX96) public {
        ...

        (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(
            _blockTimestamp()
        );

        slot0 = Slot0({
            sqrtPriceX96: sqrtPriceX96,
            tick: tick,
            observationIndex: 0,
            observationCardinality: cardinality,
            observationCardinalityNext: cardinalityNext
        });
    }
}
```

```solidity
// src/lib/Oracle.sol
library Oracle {
    ...
    function initialize(Observation[65535] storage self, uint32 time)
        internal
        returns (uint16 cardinality, uint16 cardinalityNext)
    {
        self[0] = Observation({
            timestamp: time,
            tickCumulative: 0,
            initialized: true
        });

        cardinality = 1;
        cardinalityNext = 1;
    }
    ...
}
```

### 写入观测

在 `swap` 函数中，当现价改变时，一个观测会被写入观测数组：

In `swap` function, when current price is changed, an observation is written to the observations array:

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function swap(...) public returns (...) {
        ...
        if (state.tick != slot0_.tick) {
            (
                uint16 observationIndex,
                uint16 observationCardinality
            ) = observations.write(
                    slot0_.observationIndex,
                    _blockTimestamp(),
                    slot0_.tick,
                    slot0_.observationCardinality,
                    slot0_.observationCardinalityNext
                );
            
            (
                slot0.sqrtPriceX96,
                slot0.tick,
                slot0.observationIndex,
                slot0.observationCardinality
            ) = (
                state.sqrtPriceX96,
                state.tick,
                observationIndex,
                observationCardinality
            );
        }
        ...
    }
}
```

注意到，这里观测的 tick 是 `slot0_.tick`（而不是 `state.tick`），也即这笔交易之前的价格！它在下一条语句中更新为新的价格。这正是我们之前讨论到的防价格操控机制：Uniswap 记录这个区块第一笔交易**之前**的价格，以及上一个区块最后一笔交易**之后**的价格。

另外要注意到，每个观测都通过 `_blockTimestamp()` 来标识，也即现在区块的时间戳。这意味着如果现在区块已经存在一个观测了，那么价格将不会被记录。如果当前区块没有观测（也即这是本区快中的第一笔 Uniswap 交易），价格才会被记录。这也是防价格操控机制的一部分。

```solidity
// src/lib/Oracle.sol
function write(
    Observation[65535] storage self,
    uint16 index,
    uint32 timestamp,
    int24 tick,
    uint16 cardinality,
    uint16 cardinalityNext
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    if (last.timestamp == timestamp) return (index, cardinality);

    if (cardinalityNext > cardinality && index == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }

    indexUpdated = (index + 1) % cardinalityUpdated;
    self[indexUpdated] = transform(last, timestamp, tick);
}
```

这里我们看到，如当前区块已经存在一个观测，那么将会跳过写。如果现在没有存在这样的关泽，我们将会存储一个新的，并且在可能的时候尝试扩展基数。取模运算符(`%`)确保了观测的下标保持在 $[0, cardinality)$ 区间中，并且当达到上界时重置为0。

接下来，我们来看一下 `transform` 函数：

```solidity
function transform(
    Observation memory last,
    uint32 timestamp,
    int24 tick
) internal pure returns (Observation memory) {
    uint56 delta = timestamp - last.timestamp;

    return
        Observation({
            timestamp: timestamp,
            tickCumulative: last.tickCumulative +
                int56(tick) *
                int56(delta),
            initialized: true
        });
}
```

在这里，我们计算的是累积价格：现在的 tick 乘以上次观测到现在的秒数，并加到上一个累积价格之上。

### 增加基数

现在，让我们来看一下基数是如何扩展的。

任何人在任何时间都可以扩展一个池子中观测的基数，并为此支付 gas。因此，我们需要在池子合约中增添一个新的 public 的函数：

```solidity
// src/UniswapV3Pool.sol
function increaseObservationCardinalityNext(
    uint16 observationCardinalityNext
) public {
    uint16 observationCardinalityNextOld = slot0.observationCardinalityNext;
    uint16 observationCardinalityNextNew = observations.grow(
        observationCardinalityNextOld,
        observationCardinalityNext
    );

    if (observationCardinalityNextNew != observationCardinalityNextOld) {
        slot0.observationCardinalityNext = observationCardinalityNextNew;
        emit IncreaseObservationCardinalityNext(
            observationCardinalityNextOld,
            observationCardinalityNextNew
        );
    }
}
```

以及 Oracle 合约中的一个新函数：

```solidity
// src/lib/Oracle.sol
function grow(
    Observation[65535] storage self,
    uint16 current,
    uint16 next
) internal returns (uint16) {
    if (next <= current) return current;

    for (uint16 i = current; i < next; i++) {
        self[i].timestamp = 1;
    }

    return next;
}
```

在 `grow` 函数中，我们通过把 `timestamp` 值设置为一些非零值来分配这些新的观测。注意到，`self` 是一个 `storage` 的变量，为它的元素分配值会更新数组计数器，并且把这些值写道合约的存储中。

### 读取观测

We've finally come to the trickiest part of this chapter: reading of observations. Before moving on, let's review how
observations are stored to get a better picture.

Observations are stored in a fixed-length array that can be expanded:

![Observations array](/images/milestone_5/observations.png)

As we noted above, observations are expected to overflow: if a new observation doesn't fit into the array, writing
continues starting at index 0, i.e. oldest observations get overwritten:

![Observations wrapping](/images/milestone_5/observations_wrapping.png)

There's no guarantee that an observation will be stored for every block because swaps don't happen in every block. Thus,
there will be blocks we don't know prices at, and such periods of missing observations can be long. Of course, we don't
want to have gaps in the prices reported by the oracle, and this is why we're using time-weighted average prices (TWAP)–so we
could have averaged prices in the periods where there were no observations. TWAP allows us to *interpolate* prices, i.e.
to draw a line between two observations–each point on the line will be a price at a specific timestamp between the two
observations.

![Interpolated prices](/images/milestone_5/interpolated_prices.png)


So, reading observations means finding observations by timestamps and interpolating missing observations, taking into
consideration that the observations array is allowed to overflow (e.g. the oldest observation can come after the most
recent one in the array). Since we're not indexing the observations by timestamps (to save gas), we'll need to use the
[binary search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm) to efficient search. But not always.

Let's break it down into smaller steps and begin by implementing `observe` function in `Oracle`:

```solidity
function observe(
    Observation[65535] storage self,
    uint32 time,
    uint32[] memory secondsAgos,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56[] memory tickCumulatives) {
    tickCumulatives = new int56[](secondsAgos.length);

    for (uint256 i = 0; i < secondsAgos.length; i++) {
        tickCumulatives[i] = observeSingle(
            self,
            time,
            secondsAgos[i],
            tick,
            index,
            cardinality
        );
    }
}
```

The function takes current block timestamp, the list of time points we want to get prices at (`secondsAgo`), current
tick, observations index, and cardinality.

Moving to the `observeSingle` function:

```solidity
function observeSingle(
    Observation[65535] storage self,
    uint32 time,
    uint32 secondsAgo,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56 tickCumulative) {
    if (secondsAgo == 0) {
        Observation memory last = self[index];
        if (last.timestamp != time) last = transform(last, time, tick);
        return last.tickCumulative;
    }
    ...
}
```

When most recent observation is requested (0 seconds passed), we can return it right away. If it wasn't record in the
current block, transform it to consider the current block and the current tick.

If an older time point is requested, we need to make several checks before switching to the binary search algorithm:
1. if the requested time point is the last observation, we can return the accumulated price at the latest observation;
1. if the requested time point is after the last observation, we can call `transform`  to find the accumulated price at
this point, knowing the last observed price and the current price;
1. if the requested time point is before the last observation, we have to use the binary search.

Let's go straight to the third point:
```solidity
function binarySearch(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    uint16 index,
    uint16 cardinality
)
    private
    view
    returns (Observation memory beforeOrAt, Observation memory atOrAfter)
{
    ...
```

The function takes the current block timestamp (`time`), the timestamp of the price point requested (`target`), as well
as the current observations index and cardinality. It returns the range between two observations in which the requested time
point is located.

To initialize the binary search algorithm, we set the boundaries:
```solidity
uint256 l = (index + 1) % cardinality; // oldest observation
uint256 r = l + cardinality - 1; // newest observation
uint256 i;
```

Recall that the observations array is expected to overflow, that's why we're using the modulo operator here.

Then we spin up an infinite loop, in which we check the middle point of the range: if it's not initialized (there's no
observation), we're continuing with the next point:

```solidity
while (true) {
    i = (l + r) / 2;

    beforeOrAt = self[i % cardinality];

    if (!beforeOrAt.initialized) {
        l = i + 1;
        continue;
    }

    ...
```

If the point is initialized, we call it the left boundary of the range we want the requested time point to be included
in. And we're trying to find the right boundary (`atOrAfter`):

```solidity
    ...
    atOrAfter = self[(i + 1) % cardinality];

    bool targetAtOrAfter = lte(time, beforeOrAt.timestamp, target);

    if (targetAtOrAfter && lte(time, target, atOrAfter.timestamp))
        break;
    ...
```
If we've found the boundaries, we return them. If not, we continue our search:

```solidity
    ...
    if (!targetAtOrAfter) r = i - 1;
    else l = i + 1;
}
```

After finding a range of observations the requested time point belongs to, we need to calculate the price at the
requested time point:
```solidity
// function observeSingle() {
    ...
    uint56 observationTimeDelta = atOrAfter.timestamp -
        beforeOrAt.timestamp;
    uint56 targetDelta = target - beforeOrAt.timestamp;
    return
        beforeOrAt.tickCumulative +
        ((atOrAfter.tickCumulative - beforeOrAt.tickCumulative) /
            int56(observationTimeDelta)) *
        int56(targetDelta);
    ...
```

This is as simple as finding the average rate of change within the range and multiplying it by the number of seconds
that has passed between the lower bound of the range and the time point we need. This is the interpolation we discussed
earlier.

The last thing we need to implement here is a public function in Pool contract that reads and returns observations:

```solidity
// src/UniswapV3Pool.sol
function observe(uint32[] calldata secondsAgos)
    public
    view
    returns (int56[] memory tickCumulatives)
{
    return
        observations.observe(
            _blockTimestamp(),
            secondsAgos,
            slot0.tick,
            slot0.observationIndex,
            slot0.observationCardinality
        );
}
```

### Interpreting Observations

Let's now see how to interpret observations.

The `observe` function we just added returns an array of accumulated prices, and we want to know how to convert them
to actual prices. I'll demonstrate this in a test of the `observe` function.

In the test, I run multiple swaps in different directions and at different blocks:

```solidity
function testObserve() public {
    ...
    pool.increaseObservationCardinalityNext(3);

    vm.warp(2);
    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);

    vm.warp(7);
    pool.swap(address(this), true, swapAmount2, sqrtP(4000), extra);

    vm.warp(20);
    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);
    ...
```

> `vm.warp` is a cheat-code provided by Foundry: it forwards to a block with the specified timestamp. 2, 7, 20 – these
are block timestamps.

The first swap is made at the block with timestamp 2, the second one is made at timestamp 7, and the third one is made at
timestamp 20. We can then read the observations:

```solidity
    ...
    secondsAgos = new uint32[](4);
    secondsAgos[0] = 0;
    secondsAgos[1] = 13;
    secondsAgos[2] = 17;
    secondsAgos[3] = 18;

    int56[] memory tickCumulatives = pool.observe(secondsAgos);
    assertEq(tickCumulatives[0], 1607059);
    assertEq(tickCumulatives[1], 511146);
    assertEq(tickCumulatives[2], 170370);
    assertEq(tickCumulatives[3], 85176);
    ...
```

1. The earliest observed price is 0, which is the initial observation that's set when the pool is deployed. However,
since the cardinality was set to 3 and we made 3 swaps, it was overwritten by the last observation.
1. During the first swap, tick 85176 was observed, which is the initial price of the pool–recall that the price before
a swap is observed. Because the very first observation was overwritten, this is the oldest observation now.
1. Next returned accumulated price is 170370, which is `85176 + 85194`. The former is the previous accumulator value,
the latter is the price after the first swap that was observed during the second swap.
1. Next returned accumulated price is 511146, which is `(511146 - 170370) / (17 - 13) = 85194`, the accumulated price
between the second and the third swap.
1. Finally, the most recent observation is 1607059, which is `(1607059 - 511146) / (20 - 7) = 84301`, which is ~4581
USDC/ETH, the price after the second swap that was observed during the third swap.

And here's an example that involves interpolation: the time points requested are not the time points of the swaps:

```solidity
secondsAgos = new uint32[](5);
secondsAgos[0] = 0;
secondsAgos[1] = 5;
secondsAgos[2] = 10;
secondsAgos[3] = 15;
secondsAgos[4] = 18;

tickCumulatives = pool.observe(secondsAgos);
assertEq(tickCumulatives[0], 1607059);
assertEq(tickCumulatives[1], 1185554);
assertEq(tickCumulatives[2], 764049);
assertEq(tickCumulatives[3], 340758);
assertEq(tickCumulatives[4], 85176);
```

This results in prices: 4581.03, 4581.03, 4747.6, 5008.91, which are the average prices within the requested intervals.

> Here's how to compute those values in Python:
> ```python
> vals = [1607059, 1185554, 764049, 340758, 85176]
> secs = [0, 5, 10, 15, 18]
> [1.0001**((vals[i] - vals[i+1]) / (secs[i+1] - secs[i])) for i in range(len(vals)-1)]
> ```