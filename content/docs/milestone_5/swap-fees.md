---
title: "交易费率"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 交易费率

正如我在简介中所说，交易费率是 Uniswap 的一个核心机制。LP 需要从提供流动性中获得收益，否则它们还不如把这笔钱用来做别的。为了激励它们，每一笔交易中交易员都会付出一小笔费用。这些费用将会按提供的流动性占比分配给所有的 LP。

为了更好地理解费用收集和分发的机制，我们来看一下它如何工作。

## 费率如何收集

![Liquidity ranges and fees](/images/milestone_5/liquidity_ranges_fees.png)

交易费用仅仅当一个价格区间在使用中的时候才会被收集到这个区间中。因此，我们需要跟踪穿过价格区间边界的时间。我们希望在下列的时间开始对一个价格区间的费率收集：
1. 当价格上升，tick 穿过价格区间的下界；
2. 当价格下降，tick 穿过价格区间的上界；

而在下列的时刻一个价格区间被停用：
1. 当价格上升，tick 穿过价格区间的上界；
2. 当价格下降，tick 穿过价格区间的下界；


![Liquidity range engaged/disengaged](/images/milestone_5/liquidity_range_engaged.png)

除了知道何时一个区间会被激活/停用以外，我们还希望能够跟踪每个价格区间累积了多少费用。

为了让费用计算更简单，Uniswap V3 跟踪**一个单位的流动性产生的总费用**。之后，价格区间的费用通过总费用计算出来：用总费用减去价格区间之外累计的费用。而在一个价格区间之外累积的费用是当一个 tick 被穿过时追踪的（当交易移动价格时，tick 被穿过；费用在交易中累计）。用这种方法，我们不需要去在每一笔交易中更新每一个位置累计的费用——这会节省大量的 gas，让与池子的交互更加便宜。

让我们在进行下一步之前再来回顾一下：
1. 用户交易 token 的时候支付费用。输入 token 中的一小部分将会被减去，并累积到池子的余额中。
2. 每个池子都有 `feeGrowthGlobal0X128` 和 `feeGrowthGlobal1X128` 两个状态变量，来跟踪每单位的流动性累计的总费用（也即，总的费用除以池子流动性）。
3. 注意到，此时实际的位置信息并没有更新，以便于节省 gas。
4. tick 跟踪在它之外累积的费用。当添加一个新的位置并激活一个 tick 的时候（添加流动性到一个之前是空着的 tick），这个 tick 记录在它之外累计的费用（惯例来说，我们假设之前所有积累的费用都 **低于这个 tick**）。
5. 每当一个 tick 被激活时，在这个 tick 之外积累的费用就会更新为，在这个 tick 之外积累的总费用减去上一次被穿过时这个 tick 记录的费用。
6. tick 知道了在他之外累积了多少费用，就可以让我们计算出在一个 position 内部累积了多少费用（position 就是两个 tick 之间的区间）。
7. 知道了一个 position 内部累积了多少费用，我们就能够计算 LP 能够分成到多少费用。如果一个 position 没有参与到交易中，它的累计费率会是 0，在这个区间提供流动性的 LP 将不会获得任何利润。

现在，我们来看一下如何计算一个 position 累积的费用（第六步）。

## 计算 Position 累积费用

为了计算一个 position 累计的总费用，我们需要考虑两种情况：当现价在这个区间内或者现价在区间外。在两种情况中，我们都会从总价中减去区间下界和上界之外累积的费用来获得结果。但是根据现价情况的不同，我们对于这些费用的计算方法也不同。

当现价在这个区间内，我们减去到目前为止，这些 tick 之外累积的费用：

![Fees accrued inside and outside of a price range](/images/milestone_5/fees_inside_and_outside_price_range.png)

当现价在区间之外，我们需要在减去上下界之外的费用之前先对它们进行更新。我们仅仅在计算中更新它们，而不会覆盖它们，因为这些 tick 还没有被穿过。

tick 之外累计的费用更新如下：

$$f_{o}(i) = f_{g} - f_{o}(i)$$

在 tick 之外收集的费用（$f_{o}(i)$）是总费用（$f_{g}$）与上一次这个 tick 被穿过时累计的费用之差。约等于我们在 tick 被穿过时重置一下其计数器。

计算一个 position 内累积的费用：

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

我们从所有价格区间累积的总费用中，减去在下界之下累积的费用（$f_{b}(i_{l})$）和在上界之上累计的费用（$f_{a}(i_{u})$）。也即我们上面图中看到的计算方法。

现在，当现价高于区间下界时（即区间被激活时），我们不会更新低于下界的费用累积，仅仅从下界中读取这个数据；对上界也是同理。而在另外两种情况时，我们需要考虑更新费用：
1. 当现价低于下界 tick，并考虑低于下界累积的费用时；
2. 当现价高于上界 tick，并考虑高于上界累积的费用时。

希望上面这些不会让你迷惑。幸运的是，现在我们已经理解了一切，并且可以开始写代码了！

## 累积交易费用

为了简单起见，我们会逐步在我们的代码中添加费率。我们首先从收取交易费用开始。

### 添加需要的状态变量

我们需要做的第一件事是在池子中添加费率参数——每个池子都有一个固定且不可变的费率，在部署时配置。在前一章中，我们添加了工厂合约来简化池子的部署。池子部署参数中的一个是 tick 间隔。现在，我们将会把这个参数替换成费率，并且我们会把费率和 tick 间隔绑定：费率越高，tick 间隔越大。这是因为稳定性越高的池子（稳定币池）的费率应该更低。

让我们更新工厂合约：
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    ...
    mapping(uint24 => uint24) public fees; // `tickSpacings` replaced by `fees`

    constructor() {
        fees[500] = 10;
        fees[3000] = 60;
    }

    function createPool(
        address tokenX,
        address tokenY,
        uint24 fee
    ) public returns (address pool) {
        ...
        parameters = PoolParameters({
            factory: address(this),
            token0: tokenX,
            token1: tokenY,
            tickSpacing: fees[fee],
            fee: fee
        });
        ...
    }
}
```

费率的单位是基点的百分之一，也即一个费率单位是 0.0001%，500 是 0.05%，3000 是 0.3%。

下一步是在池子中累积交易费用。为此我们要添加两个全局费用累积的变量：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    uint24 public immutable fee;
    uint256 public feeGrowthGlobal0X128;
    uint256 public feeGrowthGlobal1X128;
}
```

带 0 的那个跟踪 `token0` 累积的费用，带 1 的跟踪 `token1` 累积的费用。

### 收集费用

现在我们需要更新 `SwapMath.computeSwapStep`——这是我们计算交易数量的函数，同时也是我们计算和减去交易费用的地方。在这里，我们把所有的 `amountRemaining` 替换为 `amountRemainingLessFee`：

```solidity
uint256 amountRemainingLessFee = PRBMath.mulDiv(
    amountRemaining,
    1e6 - fee,
    1e6
);
```

这样，我们就在输入的 token 中减去了交易费用，并且用这个小一点的结果计算输出数量。

这个函数现在也会返回在这一步中累计的交易费用——它的计算方法根据是否达到了区间的上界而有所不同：

```solidity
bool max = sqrtPriceNextX96 == sqrtPriceTargetX96;
if (!max) {
    feeAmount = amountRemaining - amountIn;
} else {
    feeAmount = Math.mulDivRoundingUp(amountIn, fee, 1e6 - fee);
}
```

如果没有达到上界，现在的价格区间有足够的流动性来填满交易，因此我们只需要返回填满交易所需数量与实际数量之间的差即可。注意到，这里没有使用 `amountRemainingLessFee`，因为实际上的费用已经在重新计算 `amountIn` 的过程中考虑过了（译者注：此处建议参考[对应代码片段](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_4...milestone_5)更清晰）。

当目标价格已经达到，我们不能从整个 `amountRemaining` 中减去费用，因为现在价格区间的流动性不足以完成交易。因此，在这里的费用仅考虑这个价格区间实际满足的交易数量（`amountIn`）。

在 `SwapMath.computeSwapStep` 返回值后，我们需要更新这步交易累计的费用。注意到仅仅有一个变量来跟踪数值，这是因为当关注一笔交易的时候，我们已经知道了输入 token 是 `token0` 还是 `token1`（而不会是两者均有）：

```solidity
SwapState memory state = SwapState({
    ...
    feeGrowthGlobalX128: zeroForOne
        ? feeGrowthGlobal0X128
        : feeGrowthGlobal1X128
});

(...) = SwapMath.computeSwapStep(...);

state.feeGrowthGlobalX128 += PRBMath.mulDiv(
    step.feeAmount,
    FixedPoint128.Q128,
    state.liquidity
);
```

这里我们用费用除以流动性的数量，为了让后面在 LP 之间分配利润更加公平。

### 在 tick 中更新费用追踪器

接下来，我们需要在 tick 中更新费用追踪器（当交易中穿过一个 tick 时）：

```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(
        step.nextTick,
        (
            zeroForOne
                ? state.feeGrowthGlobalX128
                : feeGrowthGlobal0X128
        ),
        (
            zeroForOne
                ? feeGrowthGlobal1X128
                : state.feeGrowthGlobalX128
        )
    );
    ...
}
```

由于我们此时还没有更新 `feeGrowthGlobal0X128/feeGrowthGlobal1X128` 状态变量，我们把 `state.feeGrowthGlobalX128` 作为其中一个参数传入。`cross` 函数更新费用追踪器：

```solidity
// src/lib/Tick.sol
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal returns (int128 liquidityDelta) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 =
        feeGrowthGlobal0X128 -
        info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 =
        feeGrowthGlobal1X128 -
        info.feeGrowthOutside1X128;
    liquidityDelta = info.liquidityNet;
}
```

> 我们还没有添加对于 `feeGrowthOutside0X128/feeGrowthOutside1X128` 变量的初始化——我们会在后面步骤中完成。

### 更新全局费用追踪器

最后一步，当交易完成时，我们需要更新全局的费用追踪：

```solidity
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
}
```

同样地，在一笔交易中只有一个变量会更新，因为交易费仅从输入 token 中收取。

现在交易部分就完成了！现在我们来看看，当添加流动性时费用会发生什么变化。

## 位置管理中的费用

当添加或移除流动性的时候（后者我们还没有实现），我们也需要初始化或者更新费用。费用在 tick 中（在 tick 之外累计的数量，`feeGrowthOutside`）和在 position 中（position内部累积的费用）都需要进行跟踪。在 position 中，我们也需要跟踪和更新收集的费用数量——或者换句话说，我们把每单位流动性的费用转换成 token 数量。因为当 LP 移除流动性的时候，它们需要获得一定数量的交易费用。

我们来一步一步完成它。

### tick 中费用追踪器的初始化

在 `Tick.update` 函数中，当一个 tick 被初始化时（添加流动性到一个空的 tick），我们初始化它的费用追踪器。然而，我们仅当 tick 低于现价的时候做这件事，也即当现价在现在价格区间内时：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 currentTick,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    bool upper
) internal returns (bool flipped) {
    ...
    if (liquidityBefore == 0) {
        // by convention, assume that all previous fees were collected below
        // the tick
        if (tick <= currentTick) {
            tickInfo.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            tickInfo.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
        }

        tickInfo.initialized = true;
    }
    ...
}
```

如果现价不在价格区间内，费用追踪器将被设置为0，并且会在下一次这个 tick 被穿过时进行更新（参考我们上面写的 `cross` 函数）。

### 更新 position 费用和 token 数量

下一步是计算 position 累计的费用和 token 数量。由于一个 position 就是两个 tick 之间的一个区间，我们使用 tick 中的费用追踪器来计算这些值。下面这个函数可能看起来有点复杂，但它实现的正是我们之前看到的价格区间费用计算的公式：

```solidity
// src/lib/Tick.sol
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 lowerTick_,
    int24 upperTick_,
    int24 currentTick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    Tick.Info storage lowerTick = self[lowerTick_];
    Tick.Info storage upperTick = self[upperTick_];

    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (currentTick >= lowerTick_) {
        feeGrowthBelow0X128 = lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lowerTick.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside1X128;
    }

    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (currentTick < upperTick_) {
        feeGrowthAbove0X128 = upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upperTick.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 =
        feeGrowthGlobal0X128 -
        feeGrowthBelow0X128 -
        feeGrowthAbove0X128;
    feeGrowthInside1X128 =
        feeGrowthGlobal1X128 -
        feeGrowthBelow1X128 -
        feeGrowthAbove1X128;
}
```

这里我们计算两个 tick 之间累计的费用。首先我们计算低于下界 tick 的费用，然后是高于上界 tick 的费用。在最后，我们把这些费用从全局积累的费用中减去。这正是我们之前看到的公式：

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

当计算在某个 tick 之上/之下累积的费用时，我们根据当前价格区间是否被激活（现价是否在价格区间内）来进行不同操作。当它处于活跃状态，我们只需要使用当前 tick 的费用追踪器的值；当它处于停用Z黄台，我们需要使用 tick 更新后的费用——你可以在上面代码里两个 `else` 分支的计算中看到。

得到 position 内累积的费用后，我们可以更新 position 内的费用和数量追踪器了：

```solidity
// src/lib/Position.sol
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    uint128 tokensOwed0 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside0X128 - self.feeGrowthInside0LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );
    uint128 tokensOwed1 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside1X128 - self.feeGrowthInside1LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );

    self.liquidity = LiquidityMath.addLiquidity(
        self.liquidity,
        liquidityDelta
    );
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;

    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

当计算应得的 token 时，我们把费用乘以区间的流动性——与我们在交易时所作的相反。在最后，我们更新费用追踪器，并把 token 数量加到之前的数量上。

现在，每当一个 position 发生变动（添加或移除流动性），我们计算这个区间收集的费用并且更新 position 信息：

```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    bool flippedLower = ticks.update(params.lowerTick, ...);
    bool flippedUpper = ticks.update(params.upperTick, ...);
    ...
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) = ticks
        .getFeeGrowthInside(
            params.lowerTick,
            params.upperTick,
            slot0_.tick,
            feeGrowthGlobal0X128_,
            feeGrowthGlobal1X128_
        );

    position.update(
        params.liquidityDelta,
        feeGrowthInside0X128,
        feeGrowthInside1X128
    );
    ...
}
```

## 移除流动性

我们现在可以来添加我们唯一一个没有实现的核心功能了——移除流动性。与 `mint` 相对应，我们把这个函数叫做 `burn`。这个函数允许 LP 移除一个 position 中部分或者全部的流动性。除此之外，它也会计算 LP 应该得到的利润收入。然而，实际的 token 转移会在另一个函数中实现——`collect`。

### 燃烧流动性

燃烧流动性与铸造相反。我们现在的设计和实现使得这个任务非常简单——燃烧流动性仅仅是符号为负的铸造。它就等同于添加负数的流动性。

> 为了实现 `burn`，我们需要重构代码，把 position 管理相关的代码（更新 tick 和 position，以及 token 数量的计算）移动到 `_modifyPosition` 函数中，这个函数会被 `mint` 和 `burn` 使用。


```solidity
function burn(
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) public returns (uint256 amount0, uint256 amount1) {
    (
        Position.Info storage position,
        int256 amount0Int,
        int256 amount1Int
    ) = _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                lowerTick: lowerTick,
                upperTick: upperTick,
                liquidityDelta: -(int128(amount))
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, lowerTick, upperTick, amount, amount0, amount1);
}
```

在 `burn` 函数中，我们首先更新 position，并从中移除一定数量的流动性。接下来，我们更新这个 position 应得的 token 数量——它包含提供流动性时转入的 token 数量以及费用收入。我们也可以把它看做把 position 流动性转换到 token的过程——这些 token 将不会再被用于流动性，并且可以通过调用 `collect` 函数来赎回：

```solidity
function collect(
    address recipient,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount0Requested,
    uint128 amount1Requested
) public returns (uint128 amount0, uint128 amount1) {
    Position.Info memory position = positions.get(
        msg.sender,
        lowerTick,
        upperTick
    );

    amount0 = amount0Requested > position.tokensOwed0
        ? position.tokensOwed0
        : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1
        ? position.tokensOwed1
        : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        IERC20(token0).transfer(recipient, amount0);
    }

    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        IERC20(token1).transfer(recipient, amount1);
    }

    emit Collect(
        msg.sender,
        recipient,
        lowerTick,
        upperTick,
        amount0,
        amount1
    );
}
```

这个函数仅仅是从池子中转出 token，并确保只能转出有效的数量（不能够转出超过燃烧+小费收入的数量）。

这种方式也可以在不燃烧流动性的情况下取出费用收入：燃烧流动性数量设置为 0，然后调用 `collect`。在燃烧过程中，position 会被更新，应得的 token 数量也会更新。

就是这样！我们的池子实现现在已经完成了！
