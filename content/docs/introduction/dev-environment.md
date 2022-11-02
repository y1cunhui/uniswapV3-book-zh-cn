---
title: "开发环境"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 开发环境

本书中，我们会搭建两个应用：
1. 链上：一套部署在以太坊上的智能合约
2. 链下：一个与智能合约交互的前端应用

尽管本书把前端应用作为其中一部分，但这不是我们的主要关注点。我们搭建前端仅仅为了展示智能合约是如何集成到前端应用中的。因为，前端部分是选读内容，但在本书代码仓库中也提供了这部分的代码。

## 以太坊简介

以太坊是一个允许任何人在上面运行应用的区块链。它与一个传统云服务的主要区别在于：
1. 维持这个应用不需要付费，部署应用需要付费；
2. 应用代码是不可变的，在其部署后你没有办法再进行修改；
3. 用户调用你的应用需要花费 gas（手续费）；

为了更好地理解这些区别，我们来看一下以太坊的构成。

以太坊的核心（其他区块链也是同理）是一个数据库。这个数据库中最有价值的数据是*账户状态*。以太坊中的每个账户包含一个地址，以及以下数据：
1. 余额：账户的以太坊(ether)余额
2. 代码：部署在这个地址上的智能合约的字节码
3. 存储：智能合约存储数据的空间
4. nonce：一个用来防止重放攻击的整数序号

（译者注：Ethereum 和 ether 的中文译名均为以太坊，读者注意区分。Ethereum 指的是这条区块链，ether 指的是该链上的原生代币 ETH）

以太坊的主要任务是安全地维护这些数据，防止它们被未经授权的操作更改。

同时，以太坊也是一个网络，网络中的每个计算机都独立地构建和维持这些状态。网络的主要目标是能够**去中心化地访问数据库**：没有任何单个权威机构可以单方面修改任何数据。这是通过叫做*共识(consensus)*的机制来实现的，即网络中节点遵守的一系列规则。如果有节点违反了规则，它将会被从这个网络中排除。

> 有趣的是，区块链也可以使用 MySQL！只是可能会存在一定的性能问题。以太坊中使用的是 [LevelDB](https://github.com/google/leveldb)，一个高效的 KV 数据库。

每个以太坊节点运行 EVM，以太坊虚拟机。虚拟机是一个能够执行其他程序的程序，EVM 则是执行智能合约程序的程序。用户通过交易(transactions)与合约交互，除了能够简单的发送 ether，交易也能够调用智能合约，需要传输的数据包括：
1. 一个合约函数的签名
2. 函数参数

交易被打包进区块，区块被矿工挖出。网络中的每个节点都可以验证每一个区块中的每一笔交易。

某种意义上来说，智能合约与 JSON APIs 有一定类似，区别就是你调用的是智能合约函数。与 API 的后端类似，智能合约也执行程序逻辑，并且也可能更改智能合约中存储的数据。而与 API 不同的是，你需要发送一笔交易来改变区块链的状态，并且你需要为每一笔交易付费。

最后，以太坊节点也实现了一套 JSON-RPC API。我们可以通过这个 API 与节点进行交互：获取账户余额、估算 gas 费、获取区块和交易、发送交易、执行不上链的智能合约调用（仅能读数据）。[在这里](https://eth.wiki/json-rpc/API)你可以获得一个可用节点的列表。


> 交易也是通过这个 API 发送的, 参见 [eth_sendTransaction](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sendtransaction).

## 本地开发环境

我们将要搭建智能合约并且在以太坊上运行它们，这意味着我们需要一个节点。在本地测试和运行合约也需要一个节点。曾经这使得智能合约开发十分麻烦，因为在一个真实节点上运行大量的测试会十分缓慢。不过现在已经有了很多快速简洁的解决方案。例如 [Truffle](https://trufflesuite.com) 和 [Hardhat](https://hardhat.org)。不过这些工具的问题在于我们需要用 JavaScript 来写测试以及与区块链的交互，这是因为 Truffle 和 Hardhat 都运行了一个本地节点，并且使用 JavaScript 的 Web3 库来与节点交互。

我们将会选择一个新的框架，Foundry。


### Foundry

[Foundry](https://github.com/foundry-rs/foundry) 是一套用于以太坊应用开发的工具包。我们将会使用以下这些工具：
1. [Forge](https://github.com/foundry-rs/foundry/tree/master/forge)，一个 Solidity 的测试框架.
2. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil)，一个本地以太坊节点。我们将会用它来部署我们的合约，并且与前端 app 交互。
3. [Cast](https://github.com/foundry-rs/foundry/tree/master/cast), 一个非常好用的 CLI 工具。


Forge 使智能合约开发更加容易。当使用 Forge 开发时，我们不需要运行一个本地节点来进行测试；Forge 会在其内置的 EVM 上运行测试：这大大加快了开发速度，让我们不再需要给节点发送交易和挖出区块。除此以外，Forge 还能够使用 Solidity 编写测试！Forge 也内置了一些机制方便我们模拟区块链的各种状态：修改某个账户余额，从其他地址执行合约，把合约部署在任意地址等等。

然而，我们仍然需要一个本地节点来部署合约。在这里我们将会使用 Anvil。前端应用可以使用 JavaScript 的库来与以太坊节点进行交互。

### Ethers.js

[Ethers.js](https://github.com/ethers-io/ethers.js/) 是一个与 Ethereum 交互的 JavaScript 库. 这是 Dapp 开发中使用的两个最流行的 JS 库之一(另一个是 [web3.js](https://github.com/ChainSafe/web3.js))。这些库让我们可以使用 JSON-API 与以太坊节点交互，并且也有各种功能丰富的函数来帮助开发者更容易地开发应用。

### MetaMask

[MetaMask](https://metamask.io/) 是一个浏览器中的以太坊钱包。它是一个浏览器插件，可以安全地创建和存储私钥。Metamask 也是最常用的以太坊钱包应用，我们将使用它来与我们的本地运行的节点进行交互。

### React

[React](https://reactjs.org/) 是前端开发中使用的一个著名的 JS 库. 本书并不要求会 React，我们将会提供一个模板。

## 准备开发环境


创建一个新的文件夹，并在其中运行 `forge init` ：
```shell
$ mkdir uniswapv3clone
$ cd uniswapv3clone
$ forge init
```

> 如果你使用 Visual Studio Code 进行开发，可以在 `forge init` 中加入 `--vscode` 这个flas：`forge init --vscode`。Forge 会在初始化时对于 VSCode 进行特别设置。

Forge会在 `src`, `test`, 和 `script` 创建样例合约，这些都可以删掉。


设置前端开发环境:
```shell
$ npx create-react-app ui
```
