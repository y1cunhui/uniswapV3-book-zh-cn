---
title: "提供流动性"
weight: 3
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}



# 提供流动性

有了这些理论，我们现在可以开始写代码了！

新建一个文件夹，cd进去运行 `forge init --vscode` 来初始化一个Forge项目。加上 `--vscode` 会让Forge配置vscode的Solidity插件。
删除其中的合约和测试文件：
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

现在，我们可以开始写我们的第一个合约了~

## 池子合约

正如我们在简介中提到的，Uniswap 部署了多个池子合约，每一个负责一对 token 的交易。Uniswap 的所有合约被分为以下两类：
- 核心合约(core contracts)
- 外部合约(periphery contracts)
  
正如其名，核心合约实现了核心的逻辑。这些合约是最小的，对用户**不友好**的，底层的合约。这些合约都只做一件事并且保证这件事尽可能地安全。在 Uniswap V3 中，核心合约包含以下两种：
1. 池子(Pool)合约，实现了去中心化交易的核心逻辑
2. 工厂(Factory)合约，作为池子合约的注册入口，使得部署池子合约更加简单。

我们将会从池子合约开始，这部分实现了 Uniswap 99% 的核心功能。

创建 `src/UniswapV3Pool.sol`:

```solidity
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```

让我们想一下这个合约需要存储哪些数据：
1. 由于每个合约都是一对 token 的交易市场，我们需要存储两个 token 的地址。这些地址是静态的，仅设置一次并且保持不变的(因此，这些变量需要被设置为 immutable)；
2. 每个池子合约包含了一系列的流动性位置，我们需要用一个 mapping 来存储这些信息，key 代表不同位置，value 是包含这些位置相关的信息；
3. 每个池子合约都包含一些 tick 的信息，需要一个 mapping 来存储 tick 的下标与对应的信息；
4. tick 的范围是固定的，这些范围在合约中存为常数；
5. 需要存储池子流动性的数量 $L$；
6. 最后，我们还需要跟踪现在的价格和对应的 tick。我们将会把他们存储在一个 slot 中来节省 gas 费：因为这些变量会被频繁读写，所以我们需要充分考虑 [Solidity 变量在存储中的分布特点](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)


总之，合约大概存储了以下这些信息：

```solidity
// src/lib/Tick.sol
library Tick {
    struct Info {
        bool initialized;
        uint128 liquidity;
    }
    ...
}

// src/lib/Position.sol
library Position {
    struct Info {
        uint128 liquidity;
    }
    ...
}

// src/UniswapV3Pool.sol
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // Pool tokens, immutable
    address public immutable token0;
    address public immutable token1;

    // Packing variables that are read together
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
    }
    Slot0 public slot0;

    // Amount of liquidity, L.
    uint128 public liquidity;

    // Ticks info
    mapping(int24 => Tick.Info) public ticks;
    // Positions info
    mapping(bytes32 => Position.Info) public positions;

    ...
```

Uniswap V3 有很多辅助的合约，`Tick` 和 `Position` 就是其中两个。`using A for B` 是Solidity的一个语言特性，能够让你用库合约 `A` 中的函数来扩展类型 `B`，这简化了对于复杂数据结构的管理方式。

> 简洁起见，我会省略掉关于 Solidity 的语法和特性的一些细节上的解释。Solidity 有非常清楚的[文档](https://docs.soliditylang.org/en/latest/)，当遇到相关问题是可以参考这里。

接下来，我们在 constructor 中初始化其中一些变量：

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```
在构造函数中，我们初始化了不可变的 token 地址、现在的价格和对应的 tick。我们暂时还不需要提供流动性。

从这里开始，到本节的最后我们会使用我们预先计算好的数值，完成我们的第一笔交易

## 铸造(Minting)

在 Uniswap V2 中，提供流动性被称作 *铸造(mint)*，因为 Uniswap V2 的池子给予 LP-token 作为提供流动性的交换。V3 没有这种行为，但是仍然保留了同样的名字，我们在这里也同样使用这个名字：

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

我们的 `mint` 函数会包含以下参数：
1. token 所有者的地址，来识别是谁提供的流动性；
2. 上界和下界的 tick，来设置价格区间的边界；
3. 希望提供的流动性的数量

> 注意到在这里，用户指定了 $L$，而不是具体的 token 数量。这显然不是特别方便，但是要记得池子合约是核心合约的一部分——它并不需要用户友好，因为它仅实现了最小的核心逻辑。在后面章节中，我们会实现一些辅助合约，来帮助用户在调用 `Pool.mint` 前将token数目转换成 $L$。

我们简单描述一下铸造函数如何工作：
1. 用户指定价格区间和流动性的数量；
2. 合约更新 `ticks` 和 `positions` 的mapping；
3. 合约计算出用户需要提供的token数量（在本节我们用事先计算好的值）；
4. 合约从用户处获得token，并且验证数量是否正确。

首先来检查 ticks：

```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```
并且确保流动性的数量不为零：

```solidity
if (amount == 0) revert ZeroLiquidity();
```

接下来，增加 tick 和 position 的信息：

```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

`ticks.update` 函数如下所示：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
```
它初始化一个流动性为 0 的 tick，并且在上面添加新的流动性。正如上面所示，我们会在下界 tick 和上界 tick 处均调用此函数，流动性在两边都有添加。


`position.update` 函数如下所示:
```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```
与 tick 的函数类似，它也在特定的位置上添加流动性。其中的 get 函数如下：

```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```
每个位置都由三个变量所确定：LP 地址，下界 tick 下标，上界 tick 下标。我们将这三个变量哈希来减少数据存储开销：哈希结果只有 32 字节，而三个变量分别存储需要 96 字节。

> 如果我们使用三个变量来标定，我们就需要三个 mapping。每个变量都分别需要 32 字节的开销，因为 solidity 会把变量存储在 32 字节的 slot 中（此处没有 packing）。

让我们继续完成我们的 mint 函数。接下来我们需要计算用户需要质押 token 的数量，幸运的是，我们在上一章中已经用公式计算出了对应的数值。在这里我们会在代码中硬编码这些数据：


```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

> 在后面的章节中，我们会把这里替换成真正的计算。

现在，我们可以从用户处获得 token 了。这部分是通过 callback 来实现的：

```solidity
uint256 balance0Before;
uint256 balance1Before;
if (amount0 > 0) balance0Before = balance0();
if (amount1 > 0) balance1Before = balance1();
IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
    amount0,
    amount1
);
if (amount0 > 0 && balance0Before + amount0 > balance0())
    revert InsufficientInputAmount();
if (amount1 > 0 && balance1Before + amount1 > balance1())
    revert InsufficientInputAmount();
```

首先，我们记录下现在的token余额。接下来我们调用caller的 `uniswapV3MintCallback` 方法。预期调用者为合约地址，因为普通用户地址无法实现 callback 函数。使用 callback 函数看起来很不用户友好，但是这能够让合约计算 token 的数量——这非常关键，因为我们无法信任用户。

调用者需要实现 `uniswapV3MintCallback` 来将 token 转给池子合约。调用 callback 函数后，我们会检查池子合约的对应余额是否发生变化，并且增量应该大于 `amount0` 和 `amount1`：这意味着调用者已经把钱转到了池子。

最后，发出一个 Mint 事件：

```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

事件(Event)是合约数据在以太坊中标定的方式，后续可以据此进行搜索。通常来说，比较好的编程习惯是在合约的状态变量发生改变时发出一个事件，这能够让前端知道这件事情发生了。事件也包含了很多有用的信息，比如：调用者的地址，对应的流动性位置，上界和下界的 tick，新的流动性数量，两种 token 的数量。这些信息会作为日志（log）存储，任何人都可以通过收集这样的日志来重放合约中的状态变动，而不需要去遍历分析所有的区块和交易。

现在我们完成了！（出乎意料的简单）。接下来到了测试部分。

## 测试

现在，我们还不知道我们的合约是否正确。在部署我们的合约之前，我们需要写一系列的测试来保证合约功能正常。正如之前所说，Forge 是一个绝妙的测试框架，能够让我们的测试十分简单。

创建一个新的测试文件：

```solidity
// test/UniswapV3Pool.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Test.sol";

contract UniswapV3PoolTest is Test {
    function setUp() public {}

    function testExample() public {
        assertTrue(true);
    }
}
```

我们来运行一下：
```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```
测试通过了！这是显然的，因为目前我们只测试了 `true` 等于 `true`。

测试合约都继承自 `forge-std/Test.sol`。这个合约实现了一些列的测试功能，我们后续会对这些功能越来越熟悉。如果你现在就想了解，可以打开 `lib/forge-std/src/Test.sol` 简单读一下。

测试合约遵循以下规则：
1. `setUp` 函数用来准备测试样例。在每个测试样例中，我们都希望有一个配置好的环境，比如合约的部署、token的铸造、池子的初始化——这些都将在 `setUp` 中完成
2. 每个测试样例以 `test` 开头，例如 `testMint()`。这能够让Forge区分出测试样例和其他的辅助函数（这里我们也可以写任何我们需要的辅助函数）。

现在我们来真正测试 minting

### 测试 token

为了测试 minting 功能，我们需要 token。这对我们来说不是个问题，因为我们在测试中能够随意部署任何合约！更进一步，Forge 能够用依赖的方式安装其他开源合约。在这里，我们需要包含铸造功能的 ERC20 合约。我们会使用 [Solmate](https://github.com/Rari-Capital/solmate) 的 ERC20 合约（Solmate 包含了一系列 gas 优化的合约），并且创建一个继承自 Solmate 合约的ERC20 合约，开放 mint 接口。

首先安装 `solmate`:
```shell
$ forge install rari-capital/solmate
```

之后，在 `test` 文件夹中创建 `ERC20Mintable.sol` 合约（因为此合约仅用于测试）：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "solmate/tokens/ERC20.sol";

contract ERC20Mintable is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```
我们的 `ERC20Mintable` 继承了 `solmate/tokens/ERC20.sol` 的所有功能，并且额外实现了一个public的 `mint` 方法，能够让我们铸造任意数量的 token。


### 测试 Minting

现在，我们可以进行 minting 的测试了。

首先，部署所有需要用到的合约：

```solidity
// test/UniswapV3Pool.t.sol
...
import "./ERC20Mintable.sol";
import "../src/UniswapV3Pool.sol";

contract UniswapV3PoolTest is Test {
    ERC20Mintable token0;
    ERC20Mintable token1;
    UniswapV3Pool pool;

    function setUp() public {
        token0 = new ERC20Mintable("Ether", "ETH", 18);
        token1 = new ERC20Mintable("USDC", "USDC", 18);
    }

    ...
```

在 `setUp` 函数中，我们部署了 token 合约，但是不需要部署池子合约。这是因为所有的测试样例都会使用同样的 token，但是可能会使用不同的池子。

为了让池子的设置更加简单清晰，我们将在另外一个函数中完成这部分工作，`setupTestCase`。它接受一系列的测试样例参数，进行池子的设置。在我们的第一个测试样例中，我们会测试成功的流动性铸造，其参数如下所示：

```solidity
function testMintSuccess() public {
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
```

1. 我们计划向池子中质押 1 ETH 和 5000 USDC；
2. 当前的 tick 为 81576，上下界的 tick 分别为 84222 和 86129（在上一章中计算的结果）；
3. 我们将会指定预先计算好的流动性 $L$ 和现价 $\sqrt{P}$；
4. 在本样例中，我们会铸造流动性（`mintLiquidity` 参数为 true ），也会在池子合约调用 callback 时转给它 token（`shouldTransferInCallback` 参数为 true）。我们并不会在每个测试中都这样做，因此我们有这些参数的设置。

接下来，我们用上述参数调用 `setUpTestCase`：


```solidity
function setupTestCase(TestCaseParams memory params)
    internal
    returns (uint256 poolBalance0, uint256 poolBalance1)
{
    token0.mint(address(this), params.wethBalance);
    token1.mint(address(this), params.usdcBalance);

    pool = new UniswapV3Pool(
        address(token0),
        address(token1),
        params.currentSqrtP,
        params.currentTick
    );

    if (params.mintLiqudity) {
        (poolBalance0, poolBalance1) = pool.mint(
            address(this),
            params.lowerTick,
            params.upperTick,
            params.liquidity
        );
    }

    shouldTransferInCallback = params.shouldTransferInCallback;
}
```
在这个函数中，我们铸造了 token，部署了池子合约。由于 `mintLiquidity` 参数设置为 true，我们会在池子中铸造初始流动性。最后，我们设置 `shouldTransferInCallback` 变量，使得在 callback 中能读到此参数：


```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (shouldTransferInCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```
在这个测试中，是测试合约提供流动性并且调用池子的 `mint` 函数，与用户（EOA）无关。测试合约会作为用户，因此它实现了 callback 函数。测试合约仅仅是个合约而已，你可以用任何你习惯的方式来编写它。

在 `testMintSuccess` 中，我们希望池子合约能够：
1. 从用户处获取正确数量的 token；
2. 创建一个关键字和流动性正确的 position；
3. 初始化我们声明的上下界 tick；
4. 有正确的 $\sqrt{P}$ 和 $L$。

我们来实现这些功能。

铸造在 `setupTestCase` 中实现，我们不需要再写一遍了。这个函数也返回了我们需要的 token 数量，我们对其进行检查：


```solidity
(uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

uint256 expectedAmount0 = 0.998976618347425280 ether;
uint256 expectedAmount1 = 5000 ether;
assertEq(
    poolBalance0,
    expectedAmount0,
    "incorrect token0 deposited amount"
);
assertEq(
    poolBalance1,
    expectedAmount1,
    "incorrect token1 deposited amount"
);
```

我们希望池子里的 token 数量与我们预先设计好的一致。同样也直接对池子进行检查：

```solidity
assertEq(token0.balanceOf(address(pool)), expectedAmount0);
assertEq(token1.balanceOf(address(pool)), expectedAmount1);
```

接下来，我们需要检查池子创建的 position。回忆一下，在 `positions` 这个 mapping 中，我们的键值是一个哈希。我们手动计算这个键值并且获得合约中对应的 position：

```solidity
bytes32 positionKey = keccak256(
    abi.encodePacked(address(this), params.lowerTick, params.upperTick)
);
uint128 posLiquidity = pool.positions(positionKey);
assertEq(posLiquidity, params.liquidity);
```

> 由于 `Position.Info` 是一个 [struct](https://docs.soliditylang.org/en/latest/types.html#structs)，它会在返回时被解构，每个 field 分别赋值给一个单独的变量。

接下来检查 ticks：

```solidity
(bool tickInitialized, uint128 tickLiquidity) = pool.ticks(
    params.lowerTick
);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);

(tickInitialized, tickLiquidity) = pool.ticks(params.upperTick);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);
```

最后，检查 $\sqrt{P}$ 和 $L$:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5602277097478614198912276234240,
    "invalid current sqrtP"
);
assertEq(tick, 85176, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

我们会发现，用 Solidity 写测试如此简单。

### 失败

显然，仅仅测试成功的场景是不够的，我们也需要构造一些失败的测试样例。在提供流动性时可能会有哪些错误点？一些提示如下：
1. 上下界 tick 太大/太小
2. 提供流动性数量为 0
3. LP 拥有的 token 数量不足

以上测试用例的编写将留作练习。代码也可以在[这个仓库](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_1/test/UniswapV3Pool.t.sol)找到。

