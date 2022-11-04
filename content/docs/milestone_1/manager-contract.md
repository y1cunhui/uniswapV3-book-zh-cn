---
title: "管理合约"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 管理合约(Manager Contract)

在部署我们的池子合约之前，仍然有一个问题需要解决。我们之前提到过，Uniswap V3 合约由两部分构成：
1. 核心合约(core contracts)实现了最核心的功能，不提供用户友好的交互接口
2. 外围合约(periphery contracts)为核心合约实现了用户友好的接口

池子合约是核心合约，它并不需要用户友好或者实现灵活功能。它需要调用者进行所有的计算并且提供合适的参数。同时，它也没有使用ERC20的 `transferFrom` 函数来从调用者处转账，而是使用了两个 callback 函数：
1. `uniswapV3MintCallback`，当铸造流动性的时候被调用
2. `uniswapV3SwapCallback`，当交易token的时候被调用

在我们的测试中，我们在测试合约中实现了这些函数。由于只有合约才能实现 callback 函数，池子合约并不能直接被普通用户(EOA)调用。这对核心合约是没问题的，但是我们接下来会解决它🙂。

我们下一步的目标是将这个池子合约部署在一个本地的区块链上，并且使用一个前端应用与其交互。因此我们需要创建一个合约，能够让非合约的地址也与池子进行交互。让我们来实现吧！

## 工作流程

下面我们描述了管理合约的功能：
1. 为了铸造流动性，我们需要 approve 对应的 token 给管理合约；
2. 我们会调用管理合约的 `mint` 函数来铸造流动性，参数为铸造需要的参数以及池子的合约地址；
3. 管理合约会调用池子的 `mint` 函数，并且会实现 `uniswapV3MintCallback`。由于我们之前的 approve，管理合约会从我们的账户中把 token 转到池子合约；
4. 为了交易 token，我们也需要 approve 对应的 token;
5. 我们会调用管理合约的 `swap` 函数，并且与 mint 过程类似，它会调用池子合约的对应函数。管理者合约会把我们的 token 转到池子中，池子进行对应的交易，然后把得到的 token 发回给我们。

综上，管理合约主要作为用户和池子之间的中介来运行。

## 向 callback 传递数据

在实现管理合约之前，我们还需要更新我们的池子合约。

管理者合约需要能够与任何一个流动性池适配，并且能够允许任何地址调用它。为了达到这一点，我们需要对 callback 进行升级：我们需要将池子的地址和用户的地址作为参数传递。下面是我们对于 `uniswapV3MintCallback` 之前的实现（测试合约中的）：


```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (transferInMintCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```

关键点：
1. 这个函数转移的 token 是属于这个测试合约的——而我们希望使用 `transferFrom` 来从管理合约的调用者出转出 token
2. 这个函数需要知道 `token0` 和 `token1`，但这两个变量会随着不同池子而变化。

想法：我们需要改变 callback 的参数，来将用户和池子的合约地址传进去。

接下来看一下 swap 的 callback：
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0 && transferInSwapCallback) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0 && transferInSwapCallback) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```

同样，它从测试合约处转钱，并且已知 `token0` 和 `token1`。

为了将这些参数传递给 callback，我们需要首先将它们传递给 `mint` 和 `swap` 函数（因为callback函数是被这两个函数调用的）。然而，由于这些额外的数据并不会被上述两个函数本身使用，为了避免参数的混淆，我们会使用 [abi.encode()](https://docs.soliditylang.org/en/latest/units-and-global-variables.html?highlight=abi.encode#abi-encoding-and-decoding-functions) 来编码这些参数。

定义一下这些额外数据的结构：

```solidity
// src/UniswapV3Pool.sol
...
struct CallbackData {
    address token0;
    address token1;
    address payer;
}
...
```

接下来把编码后的数据传入 callback：
```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount,
    bytes calldata data // <--- New line
) external returns (uint256 amount0, uint256 amount1) {
    ...
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1,
        data // <--- New line
    );
    ...
}

function swap(address recipient, bytes calldata data) // <--- `data` added
    public
    returns (int256 amount0, int256 amount1)
{
    ...
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data // <--- New line
    );
    ...
}
```

现在，我们可以在测试合约的 callback 函数中解析对应的数据：
```solidity
function uniswapV3MintCallback(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    if (transferInMintCallback) {
        UniswapV3Pool.CallbackData memory extra = abi.decode(
            data,
            (UniswapV3Pool.CallbackData)
        );

        IERC20(extra.token0).transferFrom(extra.payer, msg.sender, amount0);
        IERC20(extra.token1).transferFrom(extra.payer, msg.sender, amount1);
    }
}
```

尝试自己动手更新代码的其余部分。如果觉得有困难，可以参考[此处](https://github.com/Jeiwan/uniswapv3-code/commit/cda23134fd12a190aaeebe718786545621e16c0e)

## 实现管理合约

除了实现 callback 函数之外，管理合约其实没什么别的功能：它仅仅是把调用重新指向池子合约。这个管理合约现在还非常非常简单：


```solidity
pragma solidity ^0.8.14;

import "../src/UniswapV3Pool.sol";
import "../src/interfaces/IERC20.sol";

contract UniswapV3Manager {
    function mint(
        address poolAddress_,
        int24 lowerTick,
        int24 upperTick,
        uint128 liquidity,
        bytes calldata data
    ) public {
        UniswapV3Pool(poolAddress_).mint(
            msg.sender,
            lowerTick,
            upperTick,
            liquidity,
            data
        );
    }

    function swap(address poolAddress_, bytes calldata data) public {
        UniswapV3Pool(poolAddress_).swap(msg.sender, data);
    }

    function uniswapV3MintCallback(...) {...}
    function uniswapV3SwapCallback(...) {...}
}
```

callback 函数与上述测试合约中的相同，除了没有 `transferInMintCallback` 和 `transferInSwapCallback` 这两个 flag，因为管理合约总会转钱。

现在，我们已经彻底完成了，可以准备部署和进行前端交互了！
