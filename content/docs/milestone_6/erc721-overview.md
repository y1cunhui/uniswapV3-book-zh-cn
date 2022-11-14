---
title: "ERC721 概述"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# ERC721 概述

首先，我们先对于 [EIP-721](https://eips.ethereum.org/EIPS/eip-721) 这个定义了 NFT 合约规范的标准进行简介。

ERC721 是一个 ERC20 的变种。两者之间最主要的区别是，ERC 721 token 是*非同质化的(non-fungible)*，也即：每一个 token 都与其他 token 不同。为了区分 ERC721 token，每一个都有一个唯一的 ID，通常用来当作 token 铸造的计数器。ERC721 token 也对于所有权有额外的概念：每个 token 的拥有者都在合约中存储。这意味着每次的转移（或者授权转移）都是针对不同 token ID 的单个 token。

Uniswap V3 流动性位置与 NFT 的共同点在于非同质化：NFT 和流动性位置都是互不通用的，并且由唯一的 ID 来定位。这样的相似性让我们可以把两个概念合并起来。

ERC20 和 ERC721 最大的区别是后者由 `tokenURI` 函数。NFT 的 token，通常在 ERC721 智能合约中实现，可以与外部存储的、非链上的信息进行连接。为了把 token ID 连接到区块链外存储的图片（或者声音，或者其他东西），ERC721 定义了 `tokrnURI` 函数。这个函数返回一个指向 JSON 文件的链接，这个链接定义了 NFT token 的元数据，例如：

```json
{
    "name": "Thor's hammer",
    "description": "Mjölnir, the legendary hammer of the Norse god of thunder.",
    "image": "https://game.example/item-id-8u5h2m.png",
    "strength": 20
}
```
（这个例子来自于 [OpenZeppelin 的 ERC721 文档](https://docs.openzeppelin.com/contracts/4.x/erc721)）

这样的一个 JSON 文件定义了：一个 token 的名字、一个 collection 的描述、到对应图片的链接，以及 token 的一些属性。

事实上，我们也可以把 JSON 元数据和 token 图片存在链上。当然这会非常贵（在链上存储数据是以太坊上最贵的操作），但是我们可以通过存储模板来让它变得更便宜。一个集合内的所有 token 的元数据都相似（大部分是一样的，只有图片链接和性质会不一样），视觉效果也相近。对于后者，我们可以使用 SVG，它是一种类似于 HTML 的格式，并且 HTML 也是一种很好的模板语言。

当把 JSON 元数据和 SVG 都存储在链上时，`tokenURI` 函数将不会返回一个链接，而是直接返回通过 [data URI scheme](https://en.wikipedia.org/wiki/Data_URI_scheme#Syntax) 编码的 JSON 元数据。SVG 图片同样也是内嵌的，我们将不再需要向外部发送请求来获取 token 元数据和图片。
