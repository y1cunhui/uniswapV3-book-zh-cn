---
title: "部署合约"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 部署

我们的第一版合约已经完成了。现在，让我们来看看如何把它部署在一个本地以太坊网络上，方便我们之后用前端 app 交互。

## 选择本地网络

智能合约的开发需要运行一个本地的网络来在开发过程中进行部署和测试。这样的网络需要具有以下特点：
1. 真实的区块链。它必须是一个真实的区块链网络而不是一个模拟器，我们希望我们的合约如果能够在这样的网络上正常工作，那也一定能在主网上正常工作；
2. 速度。我们希望我们的交易能够快速被执行，这样我们能快速迭代；
3. 以太币。为了支付gas费，我们需要一些eth，因此我们希望这个网络能够允许我们生成任意数量的eth；
4. cheat code。除了提供标准的 API，我们还希望这个网络能让我们做更多的事，例如：在任何地址上部署合约，以任何地址执行交易，直接修改合约状态等等。

今天，有许多的工具能够提供这样的功能：
1. Truffle套件中的[Ganache](https://trufflesuite.com/ganache/)
2. [Hardhat](https://hardhat.org/)，一套智能合约开发环境，除了包含本地网络节点以外还有很多有用的工具
3. Foundry中的[Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil)

所有这些解决方案都能够满足我们的需求。尽管如此，项目现在都逐渐从 Ganache（最早的解决方案）迁移到 Hardhat（目前使用最广的方案），而 Foundry 也成为开发者的新宠。Foundry 也是上述三个方案中唯一使用 Solidity 来编写测试的框架（其他框架都使用 JavaScript）。除此以外，Foundry 还允许使用 Solidity 来编写部署脚本。因此，由于我们想一直使用 Solidity，我们会使用 Anvil 来运行一个本地区块链，并且使用 Solidity 编写部署脚本。

## 运行本地区块链

Anvil 不需要进行配置，我们可以直接在命令行运行：


```shell
$ anvil
                             _   _
                            (_) | |
      __ _   _ __   __   __  _  | |
     / _` | | '_ \  \ \ / / | | | |
    | (_| | | | | |  \ V /  | | | |
     \__,_| |_| |_|   \_/   |_| |_|

    0.1.0 (d89f6af 2022-06-24T00:15:17.897682Z)
    https://github.com/foundry-rs/foundry
...
Listening on 127.0.0.1:8545
```

Anvil运行一个以太坊节点，所以它实际上并不是个网络，但也没什么问题。默认配置下它会创建 10 个账户，每个有 10000 ETH。它会把这些账户和对应私钥打印在命令行，我们会使用其中一个来部署合约和与其交互。

Anvil在 `127.0.0.1:8545` 开放了 JSON-RPC API 接口——这个接口是与以太坊节点交互的主要方式。你可以在[这里](https://ethereum.org/en/developers/docs/apis/json-rpc/)找到完整的 API 文档。在这里，你可以用 curl 与其交互：

```shell
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_chainId"}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x7a69"}
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_getBalance","params":["0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x21e19e0c9bab2400000"}
```

你也可以使用 `cast`（foundry 中的另一个组件）来访问：
```shell
$ cast chain-id
31337
$ cast balance 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
10000000000000000000000
```

现在，我们来将池子合约和管理合约都部署在本地网络上。

## 第一次部署
根本上来讲，部署一个合约意味着：
1. 将源代码编译成 EVM 字节码
2. 发送一个包含这些字节码的交易
3. 新建一个地址，执行字节码中构造函数的部分，将初始化的字节码存放在该地址。这一步是由以太坊节点自动完成的，在这笔交易被打包上链时。

部署通常包含很多个步骤：准备参数，部署辅助合约，部署主合约，初始化合约等等。脚本能帮助我们自动化完成这些步骤，并且我们现在能用 Solidity 来编写脚本！

创建 `scripts/DeployDevelopment.sol` 文件，写入以下内容：
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Script.sol";

contract DeployDevelopment is Script {
    function run() public {
      ...
    }
}
```

它看起来与测试合约十分相似，唯一的差别在于它继承自 `Script` 合约而不是 `Test`。并且按照惯例，我们需要定义一个 `run` 函数作为部署脚本的主体。在 `run` 函数中，我们首先定义部署需要的参数：

```solidity
uint256 wethBalance = 1 ether;
uint256 usdcBalance = 5042 ether;
int24 currentTick = 85176;
uint160 currentSqrtP = 5602277097478614198912276234240;
```
这些正是我们之前用过的值。这里我们需要铸造 5042 个 USDC——其中 5000 个用来提供流动性，42 个用来交易。

接下来，我们定义一系列在部署过程中需要执行的交易（每一步都是独立的交易）。我们使用`startBroadcast/endBroadcast`这个 cheat code:

```solidity
vm.startBroadcast();
...
vm.stopBroadcast();
```

> 这些 cheat code 可以在 [forge的文档](https://github.com/foundry-rs/foundry/tree/master/forge#cheat-codes)中找到，我们是从继承的 `forge-std/Script.sol` 里面得到这些功能的。

在 `broadcast()` cheat code后面，或者 `startBroadcast()/stopBroadcast()` 之间的所有语句都会被转化成交易，这些交易会被发送到执行这个脚本的节点。

在这两个 cheat code 之间，我们开始真正的部署步骤。首先需要部署两种 token：


```solidity
ERC20Mintable token0 = new ERC20Mintable("Wrapped Ether", "WETH", 18);
ERC20Mintable token1 = new ERC20Mintable("USD Coin", "USDC", 18);
```
没有 token 我们就无法部署池子，因此需要先部署 token 合约。
（译者注：由于原生代币 ETH 没有 approve 功能，因此在这里使用的是 WETH。在各种金融协议中，原生 ETH 通常都被单独拿出来处理）

> 由于我们是在本地网络上进行部署，我们需要自行部署对应的 token。在主网上和公开测试网上(Ropsten, Goerli, Sepolia)，这些 token 已经被部署。因此，如果想要在这些网络上进行部署，我们需要写网络特定的部署脚本。

接下来就是部署池子合约：
```solidity
UniswapV3Pool pool = new UniswapV3Pool(
    address(token0),
    address(token1),
    currentSqrtP,
    currentTick
);
```

然后部署管理合约：
```solidity
UniswapV3Manager manager = new UniswapV3Manager();
```

最后，我们给我们自己的地址铸造一些token用来之后交易：

```solidity
token0.mint(msg.sender, wethBalance);
token1.mint(msg.sender, usdcBalance);
```

> 在Foundry脚本中，`msg.sender` 是在 `broadcast` 块中发送交易的地址。我们可以在运行脚本的时候对其进行设置。


在脚本的最后，使用 `console.log` 打印出对应的地址信息：


```solidity
console.log("WETH address", address(token0));
console.log("USDC address", address(token1));
console.log("Pool address", address(pool));
console.log("Manager address", address(manager));
```

现在我们来运行这个脚本（确保 Anvil 在另一个窗口中正在运行）：

```shell
$ forge script scripts/DeployDevelopment.s.sol --broadcast --fork-url http://localhost:8545 --private-key $PRIVATE_KEY
```

`--broadcast` 启动交易的广播。它并不是默认开启的因为并不是每一个脚本都需要发送交易。`--fork-url` 设置我们要发送交易的节点地址。`--private-key` 设置调用者的钱包：需要私钥来签名交易。我们可以选择之前 Anvil 启动时在命令行打印出来的任何一个私钥。作者选择了第一个地址和私钥。

部署需要消耗几秒钟。最后，你会看到它发送了一系列的交易。它同时也将交易收据存在了 `broadcast` 文件夹。在 Anvil 运行的窗口里，你也能看到很多行例如 `eth_sendRawTransaction`, `eth_getTransactionByHash`,
和 `eth_getTransactionReceipt` 这样的信息——在你向 Anvil 发送交易后，Forge 使用 JSON-RPC API 来检查它们的状态并且获得交易执行的结果（收据）。

恭喜！你刚刚成功部署了一个智能合约！

## 与合约交互，ABI

现在，让我们来看看我们如何与已经部署的合约交互。

每个合约都由一系列的public函数开放。以池子合约为例，包含 `mint(...)` 和 `swap(...)`。除此之外，Solidity 为每一个 public 的变量创建了 getter，所以我们也可以调用` token0()`，`token1()`，`positions()` 等等，来访问对应变量的值。然而，由于合约都被编译成字节码，**函数名在编译过程中丢失并且不存储在区块链上**。在链上，每个函数都用一个函数选择器(selector)来表示，即函数签名哈希的前 4 字节。伪代码为：

```
hash("transfer(address,address,uint256)")[0:4]
```

> EVM 使用的是 [Keccak 哈希算法](https://en.wikipedia.org/wiki/SHA-3)，标准化名字为 SHA-3。特别地，Solidity 中的哈希函数名字为`keccak256`。

根据以上信息，我们展示两种对于部署的合约进行调用的方法：一种使用 `curl` 来进行底层的调用，一种使用 `cast`。


### Token余额

我们来检查一下部署者地址种的WETH余额。这个函数的签名是 `balanceOf(address)`（在 [ERC-20](https://eips.ethereum.org/EIPS/eip-20) 标准中定义），为了计算这个函数的 ID（即选择器），我们计算哈希并取前 4 字节：


```shell
$ cast keccak "balanceOf(address)"| cut -b 1-10
0x70a08231
```

要把地址作为参数传递进去，我们把它附在函数选择器的后面（在左边 padding 到 32 个字节，因为地址在函数调用中占 32 字节）：

> 0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266


`0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266` 是我们要查看余额的地址。这是本书选择的部署者的地址，也是 Anvil 创建时的第一个地址。

接下来，我们会使用 `eth_call` JSON-RPC 方法来进行这个调用。注意到，这一步不需要发送一个交易——我们仅仅是从合约中读取数据。

```shell
$ params='{"from":"0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","to":"0xe7f1725e7734ce288f8367e1bb143e90bb3f0512","data":"0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266"}'
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_call","params":['"$params"',"latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x00000000000000000000000000000000000000000000011153ce5e56cf880000"}
```
> "to" 地址是 USDC token 的地址，它是在上一步部署脚本运行时打印出的日志里面的地址。

以太坊节点返回的结果是字节流，为了理解结果我们需要知道返回值的类型。在 `balanceOf` 函数中，返回值的类型是 `uint256`。用 `cast` 可以把结果转换成十进制然后转换成 ethers 单位：

```shell
$ cast --to-dec 0x00000000000000000000000000000000000000000000011153ce5e56cf880000| cast --from-wei
5042.000000000000000000
```

余额是正确的，我们在自己的地址中有 5042 USDC。

### 现在的 tick 和价格

上述例子展示了底层的合约调用。通常来说，你永远不会使用 `curl` 来发起调用，而是使用某个更友好的工具或库。cast 在这里也能够帮助我们简化这一过程。


我们来使用 `cast` 获得当前合约的 tick 和 price:
```shell
$ cast call POOL_ADDRESS "slot0()"| xargs cast --abi-decode "a()(uint160,int24)"
5602277097478614198912276234240
85176
```

如此容易！第一个值就是我们的 $\sqrt{P}$，第二个值就是现在的 tick。

> 由于 `--abi-decode` 需要完整的函数签名，我们必须声明一个函数名 "a()"，尽管我们只是为了解码函数的输出。


### ABI

为了简化与合约的交易，Solidity 编译器可以输出 ABI，Application Binary Interface（应用二进制接口）。

ABI 是一个包含了合约中所有 public 方法和事件的 JSON 文件。文件的目的在于使得编码函数参数和解码函数输出都更加容易。我们可以通过 Forge 来获取 ABI：


```shell
$ forge inspect UniswapV3Pool abi
```

可以看一看生成的文件，来更好地理解 ABI 意味着什么。
