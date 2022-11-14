---
title: "用户界面"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 用户界面

在引入交易路径后，我们可以大大简化我们 web 应用的内部逻辑。首先，每一个交易都使用路径，因为路径并不一定要有多个池子；其次，现在更容易改变交易方向了：我们只需要把路径翻转即可；最后，由于通过 `CREATE2` 和盐值能够确定唯一的合约地址，我们不再需要存储合约地址以及考虑 token 顺序了。

然而，在把多池子交易集成到 web 应用之前，我们还必须添加一个非常重要的算法。问自己这样一个问题：”如何找到两个没有池子的 token 之间的路径？“

## 自动路由

Uniswap 实现了一个叫做*自动路由(AutoRouter)*的算法，能够在两种 token 之间寻找最短路径。它还能够把一笔支付拆成多笔支付，来寻找到最好的平均交易价格。[比起未拆分的交易，这种交易方式的利润能够增加 36.84%](https://uniswap.org/blog/auto-router-v2)。这听起来很不错，然而，我们并不会实现一个这么高级的算法。我们只实现一个很简单的版本。

## 一个简单的路由设计

假设我们有这么一堆池子：

![Scattered pools](/images/milestone_4/pools_scattered.png)

我们如何在这一堆中寻找到两个 token 之间的最短路径呢？

这类问题最合适的解决方法是基于*图(graph)*的算法。图是一种数据结构，由节点（代表实体）和边（节点之间的联系）组成，我们可以把这一对池子转换成一个图，每个节点是一种 token，每条边是这个 token 属于的一个池子。所以一个池子在图中的表示就是由一条边项链的两个节点。上述池子可以转化成下面这样的图：

![Pools graph](/images/milestone_4/pools_graph.png)

图的最大优势在于我们能够遍历节点来寻找路径。在这里，我们将使用 [A* 算法](https://en.wikipedia.org/wiki/A*_search_algorithm)。如果有兴趣可以学习一下这个算法如何工作，但在我们的 app 里，我们使用一个实现了这个算法的库。我们使用 [ngraph.ngraph](https://github.com/anvaka/ngraph.graph) 来建图，使用 [ngraph.path](https://github.com/anvaka/ngraph.path) 来寻找最短路径（这个库实现了 A* 算法）。

在前端中，我们创建一个路径搜索器。这是一个类，在初始化时把一系列的点对转换成一张图，后续使用这个图来寻找两个 token 之间的最短路径。

```javascript
import createGraph from 'ngraph.graph';
import path from 'ngraph.path';

class PathFinder {
  constructor(pairs) {
    this.graph = createGraph();

    pairs.forEach((pair) => {
      this.graph.addNode(pair.token0.address);
      this.graph.addNode(pair.token1.address);
      this.graph.addLink(pair.token0.address, pair.token1.address, pair.tickSpacing);
      this.graph.addLink(pair.token1.address, pair.token0.address, pair.tickSpacing);
    });

    this.finder = path.aStar(this.graph);
  }

  ...
```

在构造函数中，我们创建了一个空的图，并用相连的节点进行填充。每个节点是一个 token 地址，边上有一些相关数据，即 tick 间隔——我们能够从 A* 中寻找的路径中提取这个信息。在初始化图后，我们初始化一个 A* 算法的实例。

接线来，我们实现一个函数来寻找两个 token 之间的路径，并把它转换成一个包含 token 地址和 tick 间隔的数组：

```javascript
findPath(fromToken, toToken) {
  return this.finder.find(fromToken, toToken).reduce((acc, node, i, orig) => {
    if (acc.length > 0) {
      acc.push(this.graph.getLink(orig[i - 1].id, node.id).data);
    }

    acc.push(node.id);

    return acc;
  }, []).reverse();
}
```

`this.finder.find(fromToken, toToken)` 返回一个节点序列，但并不包含边的信息（也即 tick 间隔的信息）。因此，我们调用 `this.graph.getLink(previousNode, currentNode)` 来寻找对应的边。

现在，每当用户改变输入或者输出的 token 种类时，我们可以调用 `pathFinder.findPath(token0, token1)` 来获取一条新的路径。
