---
title: "工厂合约"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 工厂合约

Uniswap 由多个离散的池子合约构成，每个池子负责一对 token 的交易。这看起来会有些问题，因为当我们想要在两个没有池子的 token 之间进行交易时——由于没有赤字，我们无法进行交易。然而，我们仍然可以进行中间交易：第一笔交易把一种 token 转换成另一种有交易对的 token，然后再把这种 token 转换成目标 token。这个路径可以更长并且有更多种的中间 token。然而，手动进行这一个操作会非常繁琐，幸运的是，我们可以在我们的智能合约中实现这个功能，让其更简便。

*工厂(Factory)*合约是一个拥有以下功能的合约：
1. 它作为池子合约的中心化注册点。在工厂中，你可以找到所有已部署的池子，对应的 token 和地址。
2. 它简化了池子合约的部署流程。EVM允许在智能合约中部署智能合约——工厂合约使用这个性质来让池子合约的部署变得十分简单。
3. 它让池子合约的地址可预测，并且能够在注册池子之前就计算出这个地址。这让池子更容易被发现。

让我们来搭建工厂合约吧！但在此之前，我们还需要学一些新东西。

## `CREATE` 和 `CREATE2` Opcodes

EVM 有两种部署合约的方式：通过 `CREATE` 或者 `CREATE2` opcode。两者之间的唯一区别试新地址如何产生：

1. `CREATE` 使用部署者账户的 `nonce` 来产生新的合约地址（伪代码如下）:
    ```
    KECCAK256(deployer.address, deployer.nonce)
    ```

    `nonce` 是一个账户特定的交易计数器。在产生合约地址过程中使用 `nonce` 会使得在其他合约或者链下app中计算合约地址变得非常困难，主要是因为想要找到对应的 nonce，我们需要扫描账户的交易历史。
2. `CREATE2` 使用一个特殊的*盐值(salt)*来产生合约地址。这是一个由开发者选择的任意序列，能够使得地址产生更加确定性（并降低碰撞概率）：
    ```
    KECCAK256(deployer.address, salt, contractCodeHash)
    ```

我们需要知道两者的区别，因为工厂在部署池子合约时使用的是 `CREATE2`，所以池子可以获得唯一并且确定性的、能够由其他合约和链下 app 计算出来的地址。在盐值方面，工厂使用这些池子的参数计算哈希：

```solidity
keccak256(abi.encodePacked(token0, token1, tickSpacing))
```

`token0` 和 `token1` 是池子里两种 token 的地址，而 `tickSpacing` 是我们下面将要讨论的内容。

## Tick 间隔

回顾一下我们在 `swap` 函数中的循环：
```solidity
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
    ...
    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(...);
    (state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath.computeSwapStep(...);
    ...
}
```
这个循环通过在一个方向上遍历来寻找拥有流动性的已初始化的 tick。然而，这个循环时非常昂贵的：如果一个 tick 离得很远，代码将会经过两个 tick 之间的所有 tick，十分消耗 gas。为了让循环更节约 gas，Uniswap 的池子有一个叫做 `tickSpacing` 的参数设定；正如其名字所示，代表两个 tick 之间的距离——距离越大，越节省 gas。

然而，tick 间隔越大，精度越低。价格波动性低的交易对（例如两种稳定币的池子）需要更高的精度，因为在这样的池子中价格移动很小；价格波动性中等和较高的交易对可以有更低的精度，因为在这样的交易对中价格移动会很大。为了处理这样的多样性，Uniswap 允许在交易对创建时设定一个 tick 间隔。Uniswap 允许部署者在下列选项中选择：10，60，200，而简单起见我们的实现中只考虑10和60。

在实际中，tick index只能够是 `tickSpacing` 的整数倍：如果 `tickSpacing` 是 10，仅有 10 的倍数才是有效的 tick index（10，20，5000，5010等，但是8，12，5001这些不可以）。然而，需要注意的是，这个限制对于现价不起作用——现价所在的 tick 仍然可以是任意的 tick，因为我们希望价格尽可能精确。`tickSpacing` 参数仅仅限制价格区间对应 tick。

因此，一个池子可以由以下参数唯一确定：
1. `token0`,
2. `token1`,
3. `tickSpacing`;

> 正如你想的那样，可以有 token 相同但是 `tickSpacing` 不同的池子存在。

工厂合约使用这组参数来作为池子的唯一定位，并且把他们作为盐值来产生池子合约地址。

> 从现在开始，我们假定所有池子的 tick 间隔为60，而在稳定币交易对中使用10。

## 工厂合约实现

在工厂合约的构造函数中，我们需要初始化支持的 tick 间隔：

```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    mapping(uint24 => bool) public tickSpacings;
    constructor() {
        tickSpacings[10] = true;
        tickSpacings[60] = true;
    }

    ...
```
> 我们的确可以让它们是常数，但在后面的 milestone 中我们会希望它是一个映射（不同的 tick 间隔会有不同的交易费）。

工厂合约中的唯一函数是 `createPool`。这个函数首先检查创建池子所需要的所有必要条件：

```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    PoolParameters public parameters;
    mapping(address => mapping(address => mapping(uint24 => address)))
        public pools;

    ...

    function createPool(
        address tokenX,
        address tokenY,
        uint24 tickSpacing
    ) public returns (address pool) {
        if (tokenX == tokenY) revert TokensMustBeDifferent();
        if (!tickSpacings[tickSpacing]) revert UnsupportedTickSpacing();

        (tokenX, tokenY) = tokenX < tokenY
            ? (tokenX, tokenY)
            : (tokenY, tokenX);

        if (tokenX == address(0)) revert TokenXCannotBeZero();
        if (pools[tokenX][tokenY][tickSpacing] != address(0))
            revert PoolAlreadyExists();
        
        ...
```

注意到这是我们第一次在代码中看到 token 的排序：
```solidity
(tokenX, tokenY) = tokenX < tokenY
    ? (tokenX, tokenY)
    : (tokenY, tokenX);
```

从现在开始，我们将永远认为池子的 token 地址是有序的，也即 `token0` 地址小于 `token1`。我们需要强制这一点来保证盐值和合约地址的计算永远一致。

> 这个改变也会影响我们在测试和部署脚本中部署 token 的方式：我们需要确保 WETH 总是 `token0`，来使得 Solidity 中的价格计算更简单（否则，我们的价格就会是分数，比如1/5000这样）。如果 WETH 在你的测试中不是 `token0`，改变一下 token 部署的顺序。

之后，我们准备部署合约需要的参数：
```solidity
parameters = PoolParameters({
    factory: address(this),
    token0: tokenX,
    token1: tokenY,
    tickSpacing: tickSpacing
});

pool = address(
    new UniswapV3Pool{
        salt: keccak256(abi.encodePacked(tokenX, tokenY, tickSpacing))
    }()
);

delete parameters;
```

这段代码看起来很奇怪，因为 `parameters` 并没有用到。Uniswap 这里使用了[控制反转(Inversion of Control)](https://en.wikipedia.org/wiki/Inversion_of_control)来在池子创建的过程中传递参数。

> 译者注：控制反转，是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度（来源 wiki）

让我们看一下更新后的池子合约构造函数：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    constructor() {
        (factory, token0, token1, tickSpacing) = IUniswapV3PoolDeployer(
            msg.sender
        ).parameters();
    }
    ..
}
```

池子合约需要部署者实现了 `IUniswapV3PoolDeployer` 接口（仅仅定义了 `parameters()` 这样一个 getter）并且在构造函数中调用它来获取参数。控制流长下面这样：
1. `Factory`：定义 `parameters` 状态变量（实现 `IUniswapV3PoolDeployer`）并在部署池子之前设置其值。
2. `Factory`：部署池子。
3. `Pool`：在构造函数中，调用部署者的 `parameters()` 函数，希望从返回值中获取参数。
4. `Factory` 调用 `delete parameters;` 来清理 `parameter` 状态变量占用的 slot 来减少 gas 开销。这仅仅是一个临时的状态变量，只在调用 `createPool()` 时有值。

在池子创建后，我们把它存储在 `pools` 映射中（这样就能够被找到），并发出一个事件：
```solidity
    pools[tokenX][tokenY][tickSpacing] = pool;
    pools[tokenY][tokenX][tickSpacing] = pool;

    emit PoolCreated(tokenX, tokenY, tickSpacing, pool);
}
```

## 池子初始化

正如你在上面代码中看到的，我们不再在池子的构造函数中设置 `sqrtPriceX96` 和 `tick`——它现在在另一个函数 `initialize` 中完成，这个函数在池子部署后调用：

```solidity
// src/UniswapV3Pool.sol
function initialize(uint160 sqrtPriceX96) public {
    if (slot0.sqrtPriceX96 != 0) revert AlreadyInitialized();

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
}
```

这是我们现在部署池子的方式：

```solidity
UniswapV3Factory factory = new UniswapV3Factory();
UniswapV3Pool pool = UniswapV3Pool(factory.createPool(token0, token1, tickSpacing));
pool.initialize(sqrtP(currentPrice));
```

## `PoolAddress` 库

现在我们来实现一个库帮助我们计算池子合约的地址。这个库只有一个函数，`computeAddress`:
```solidity
// src/lib/PoolAddress.sol
library PoolAddress {
    function computeAddress(
        address factory,
        address token0,
        address token1,
        uint24 tickSpacing
    ) internal pure returns (address pool) {
        require(token0 < token1);
        ...
```

这个函数需要知道池子的参数（用来构成盐值）和工厂合约的地址。它需要 token 事先被排序，正如上文所述。

这个函数的核心部分：
```solidity
pool = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factory,
                    keccak256(
                        abi.encodePacked(token0, token1, tickSpacing)
                    ),
                    keccak256(type(UniswapV3Pool).creationCode)
                )
            )
        )
    )
);
```

这正是 `CREATE2` 计算新合约地址的底层实现方式。我们来拆解一下：
1. 首先，我们计算盐值（`abi.encodePacked(token0, token1, tickSpacing)`）并求哈希；
2. 接下来，我们获取池子合约的代码（`type(UniswapV3Pool).creationCode`）并求哈希；
3. 然后，我们构建这样一个字节序列：`0xff`，工厂合约地址，哈希后的盐值，哈希后的池子合约代码
4. 最后求这个序列的哈希并转换成地址。

这些步骤实现了在 [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014) 中定义的地址产生方式，这也就是那个增加了 `CREATE2` 操作码的 EIP。我们来进一步看一下组成这个字节序列的值：
1. `0xff` 是在 EIP 中定义的，为了区分由 `CREATE` 和 `CREATE2` 创建的合约地址。
2. `factory` 是调用者的地址，也即我们这里的工厂合约
3. 盐值在之前提到过——用来唯一定位池子
4. 合约代码的哈希用来防止碰撞——不同的合约可以有相同的盐值，但是它们的代码哈希会不相同。

根据这样的模式，一个合约的地址由一系列唯一标识这个合约得值哈希得到，包含它的部署者、代码、和唯一参数。我们可以在任何地方使用这个函数来求出池子的地址，而不需要进行任何外部调用或者请求工厂合约。

## 简化 Manager 和 Quoter 的接口

在管理员合约和报价合约中，我们不再需要向用户请求池子地址漏洞！这使得与合约的交互更简单，因为用户不需要知道池子的地址，他们只需要知道 token。然而，用户仍然需要指定 tick 间隔，因为求池子的盐值需要它。

而且，我们也不再需要向用户请求 `zeroForOne` 参数，因为有了 token 的排序我们能够直接求出这个参数了。`zeroForOne` 在 “from token” 小于 “to token” 的时候为 true，因为池子的 `token0` 总是小于 `token1`。类似地，`zeroForOne` 在 “from token” 大于 “to token” 的时候为 false。

> 地址是哈希，哈希也是数字，所以我们能够用“大于”或者“小于”来比较地址。
