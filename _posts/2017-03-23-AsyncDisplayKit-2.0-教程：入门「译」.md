---
layout:     post
title:      AsyncDisplayKit 2.0 教程:入门「译」
subtitle:   AsyncDisplayKit Tutorial:Getting Started
date:       2017-03-23
author:     BY
header-img: img/post-bg-iWatch.jpg
catalog: true
tags:
    - iOS
    - Objective-C
    - AsyncDisplayKit
    - 开源库
---

> AsyncDisplayKit 2.0 Tutorial: Getting Started

# 前言

> "艺术是你任何能做到极致的事"

[**AsyncDisplayKit**](http://asyncdisplaykit.org/) 是一个UI框架，最初诞生于 Facebook 的 **Paper** 应用程序。它是为了解决 Paper 团队面临的核心问题之一：如何尽可能缓解主线程的压力？

现在,许多应用程序的用户体验,很大程度上依赖于持续手势和物理动画。至少,你的UI可能是依赖于某种形式的 `scrollView`。

这些类型的用户界面完全依赖于主线程，并且对主线程阻塞非常敏感。主线程阻塞将导致丢帧，降低用户的体验。

一些主线程开销较大的任务包括：

- **计算尺寸和布局**：比如  `-heightForRowAtIndexPath:`，或者在UILbel中调用 `-sizeThatFits` 以及[指数上升](http://floriankugler.com/2013/04/22/auto-layout-performance-on-ios/)的 `AutoLayout‘s`布局计算。
- **图像解码**：想要在一个 image view 中使用 `UIImage`，首先要进行解码。
- **绘图**：复杂的文本以及手动绘制渐变和阴影。
- **对象生命周期**：创建，操纵和销毁系统对象（即创建一个UIView）

当正确使用时,AsyncDisplayKit 允许您在默认情况下异步执行所有测量、布局和渲染。无需任何额外的优化，一个应用程序可以减少约一个数量级的主线程开销。

除了这些性能优势，酷炫的 AsyncDisplayKit 还为开发者提供的便利接口，用简洁的代码就能完成复杂的功能。

在这两部分 **AsyncDisplayKit 2.0** 教程中，你将掌握使用ASDK构建一个实用的和动态的应用程序的所有要素。在第一部分中，你将要学习一些在你构建应用程序时可以用到的宏观思想。在[第二部分](https://www.raywenderlich.com/124696/asyncdisplaykit-2-0-tutorial-automatic-layout)中，你将学习如何构建自己 node 的 subclass，以及如何使用ASDK强大的布局引擎。为了更好的完成本教程，你需要会使用 Xcode 以及 熟悉 Objective-C。

> **免责声明**：ASDK不兼容 [Interface Builder和AutoLayout](http://www.youtube.com/watch?v=RY_X7l1g79Q&feature=youtu.be&t=29m37s)，因此，您将不会在本教程中使用它们，虽然ASDK完全支持Swift(除了ComponentKit)，许多开发者仍在使用
Objective-C。免费App排行榜前100大多数都没有使用Swift（至少6个使用ASDK）。出于这些原因，本系列将重点介绍 Objective-C。话虽这么说，我们已经包括了一个Swift版本的实例项目。（嘴上说没有，代码还是很诚实的😂~）

# 开始

首先，[下载初始项目](https://koenig-media.raywenderlich.com/uploads/2016/12/AsyncDisplayKit-Starter-4.zip)。

该项目使用 [CocoaPods](https://cocoapods.org/) 来拉入AsyncDisplayKit。所以，在正常的 CocoaPods 体系下，打开 `RainforestStarter.xcworkspace` 而不是`RainforestStarter.xcodeproj`。

> **注意**：需要网络连接才能完成本教程。

构建并运行以查看包含 `UITableView` 动物列表的应用程序。如果你看过了代码，`AnimalTableController` 你会发现这是一个正常且熟悉的 `UITableViewController` 类。

> **注意**：确保在真机上运行本教程中的代码，而不是在模拟器中运行。

向上滑动你将看到帧数丢失引起的卡顿。你不需要启动控制台，以便能发现到这个应用程序需要在性能方面上的一些优化。

你可以通过 **AsyncDisplayKit** 的力量来解决这个问题

# ASDisplayNode 简介

`ASDisplayNode` 是ASDK的核心类，它只是一个类似于 MVC 中的 “View” 一样的`UIView` 或 `CALayer`。认识一个 node 的最佳方法是参照你已经熟悉的 `UIViews` 和 `CALayers` 之间的关系。

记住，iOS应用程序中的所有在屏幕上的显示都通过`CALayer`对象表示的。`UIViews` 创建并且拥有一个底层的 `CALayer`，并为他们添加触摸处理和其他交互功能。`UIView` 并不是 `CALayer` 的子类，而是相互环绕，扩展其功能。

![](https://koenig-media.raywenderlich.com/uploads/2016/03/view-layer-480x229.png)	

这种抽象的情况下扩展 `ASDisplayNode`：您可以将它们视为包装一个 view，就像在 view 上添加一个 layer 一样。

通常由 Node 创建的一个常规的view，其创建和配置都在行队列中执行，并且异步渲染。

![](https://koenig-media.raywenderlich.com/uploads/2016/03/node-view-layer-480x161.png)

幸运的是，用于处理 Node 的 API 对于任何使用过的 `UIViews` 或者 `CALayers` 的人来说应该异常的熟悉。所有 View 的属性都可以等效为 Node 类。你可以访问基础的 view 或者 layer 本身，就像是访问 `view.layer` 一样

# 节点容器（The Node Containers）

虽然 Node 本身提供了巨大的性能改进的可能，但真正的强大的是它们与四个容器类结合使用时产生的黑魔法。

这些类包括：

- **ASViewController**：一个 `UIViewController` 的子类，允许你提供要管理的 Node。
- **ASCollectionNode** and **ASTableNode**：Node 等效于 `UICollectionView` 和 `UITableView`，其子类实际上保留在底层。
- **ASPagerNode**:一个`ASCollectionNode`的子类，提供极好的滑动性能相比与 `UIKit` 的 `UIPageViewController` 来说。

![](https://koenig-media.raywenderlich.com/uploads/2016/03/ragecomic-480x229.png)

说得好，但真正的黑魔法来自 `ASRangeController` 这些类用于影响所包含的 Node 的行为。现在，跟着我并把你们的脑袋放空吧~

# TableNode

你要做的第一件事就是将当前 TableView 替换为 TableNode。这个没什么难度。

#### 将 TableView 替换为 TableNode

首先，进入到 `AnimalTableController.m` 。在此类中添加下面代码下面代码。

	#import <AsyncDisplayKit/AsyncDisplayKit.h>
	
这就导入了 ASDK 框架。

然后，我们继续，替换 `tableView` 的声明属性 ：

	@property  （ strong，nonatomic ） UITableView * tableView;
	
替换为 `tableNode`：
	
	@property  （ strong，nonatomic ） ASTableNode * tableNode;
	
这将导致这个类中很多地方报错，但不要慌张！

![](https://koenig-media.raywenderlich.com/uploads/2016/03/butBut-1-480x229.png)

别担心。这些错误和警告将作为你的向导，将代码转换成我们想要的。

`-viewDidLoad` 中的报错是理所当然，因为 `tableView` 已经被替换掉。我不会让你通过 `tableNode` 替换 所有的 `tableView` 实例（我的意思是，查找和替换并非那么难），但是如果你做了，你会看到：

1. 你应该为 `ASTableNode` 分配一个属性。
2. table Node  没有调用 `-registerClass:forCellReuseIdentifier:` 方法。
3. 你不能添加一个 node 到 subview

此时，你应该将 `-viewDidLoad` 中的方法替换为：

	- (void)viewDidLoad {
	  [super viewDidLoad];
	 
	  [self.view addSubnode:self.tableNode];
	  [self applyStyle];
	}

这里要注意一个有趣的情况，你调用的是 UIView 的一个 `-addSubnode:` 方法，该方法是通过 category 添加到 `UIView` 上的，等效于: 

	[self.view addSubview:self.tableNode.view];
	
接下来，修改 `-viewWillLayoutSubviews` 中的代码：

	- (void)viewWillLayoutSubviews {
	  [super viewWillLayoutSubviews];
	 
	  self.tableNode.frame = self.view.bounds;
	}
	
这样就替换用 `self.tableNode` 替换了 `self.tableView`，并且设置了 table 的 Frame

继续修改 `-applyStyle` 方法中的代码为：

	- (void)applyStyle {
	  self.view.backgroundColor = [UIColor blackColor];
	  self.tableNode.view.separatorStyle = UITableViewCellSeparatorStyleNone;
	}
	
这是唯一设置 table 的 `separatorStyle` 的一行代码。注意 tableNode 的 view 是如何访问 table 的 `separatorStyle` 属性的。`ASTableNode` 不会暴露所有`UITableView`的的属性，所以你必须通过 tableNode 底层的 `UITableView` 实例去设置 `UITableView ` 的特殊属性。

然后，在 `-initWithAnimals:` 方法中添加。

	_tableNode = [[ASTableNode alloc] initWithStyle:UITableViewStylePlain];

并且在 **return** 之前，调用：

	
	[self wireDelegation];
	
这就会在初始化 `AnimalTableController` 的时候，创建了一个 tableNode 并且调用 `-wireDelegation` 方法 设置 tableNode 的 代理。

#### 设置  TableNode 的 DataSource & Delegate

类似于 `UITableView`，`ASTableNode` 也使用 DataSource 和 Delegate 来设置本身。TableNode 的`ASTableDataSource` 和 `ASTableDelegate` protocols 非常类似于 `UITableViewDataSource` 和 `UITableViewDelegate`。

事实上，虽然他们定义了一些完全相同的方法，如 `-tableNode:numberOfRowsInSection:`，但两组协议也不完全相同，因为 `ASTableNode` 行为和`UITableView`还以所有不同的。

找到 `-wireDelegation` 方法， 并用 `tableNode` 替换 `tableView`：

	- (void)wireDelegation {
	  self.tableNode.dataSource = self;
	  self.tableNode.delegate = self;
	}
	
现在， 你会收到警告, `AnimalTableController` 实际上不符合协议。目前，`AnimalTableController` 仅遵循 `UITableViewDataSource` 和 `UITableViewDelegate`协议。在下面的章节中，我们将遵循这些协议，使我们能够使用 tableNode 的功能。

#### 遵循 ASTableDataSource

在 `AnimalTableController.m` 开头的地方找到 `AnimalTableController` 的 `DataSource` 扩展声明：

```objc
@interface AnimalTableController (DataSource)<UITableViewDataSource>
@end
```

用 `ASTableDataSource` 替换 `UITableViewDataSource`为：

	@interface AnimalTableController (DataSource)<ASTableDataSource>
	@end
	
现在，`AnimalTableController` 已经遵循了 `AnimalTableController` 协议。本就该如此了。

导航到 `AnimalTableController.m` 的底部并找到 `DataSource` category 的实现。

首先，将 `UITableViewDataSource` 的 `-tableView:numberOfRowsInSection:`方法，
更改为`ASTableDataSource` 的版本。

```objc
- (NSInteger)tableNode:(ASTableNode *)tableNode numberOfRowsInSection:(NSInteger)section {
  return self.animals.count;
}
```

接着，`ASTableNodes` 的 cells 会以不同于 `UITableView` 的方式返回。用下面的代码替换 `-tableView:cellForRowAtIndexPath:` 以适应新的规则。

```objc
// 1
- (ASCellNodeBlock)tableNode:(ASTableView *)tableView nodeBlockForRowAtIndexPath:(NSIndexPath *)indexPath {
  
  // 2
  RainforestCardInfo *animal = self.animals[indexPath.row];
 
  // 3 return ASCellNodeBlock
  return ^{
    // 4
    CardNode *cardNode = [[CardNode alloc] initWithAnimal:animal];
 
    //You'll add something extra here later...
    return cardNode;
  };
}
```
让我们整理一下：

1. ASDK 中的 `ASCellNode` 等价于 `UITableViewCell` 或者 `UICollectionViewCell`。要注意的是这个方法返回的是一个 `ASCellNodeBlock`，`ASTableNode` 维持着内部所有的 Cell，每个 indexPath 对应一个 block，并且随时准备进行初始化。
2. 你的首要任务是通过数据模型构建cell。这是非常重要的一步，要注意！你获取数据后在 下面的 block 处理。不要在 block 里引用`indexPath`,以防止 block 运行前的数据变动。
3. 然后返回一个 block，其返回值必须为 `ASCellNode`。
4. 没有必要担心Cell的复用以及初始化一个Cell的方法。您可能会注意到您现在返回了`CardNode`，而不是`CardCell`。

这让我想到一个重要的点。或许你已经了解到，**使用 ASDK 不需要复用 cell**，好吧，我已经说了两遍了，但能记住就好。请随意删除顶部`kCellReuseIdentifier`的定义吧

```objc
static NSString *kCellReuseIdentifier = @"CellReuseIdentifier";
```

你不必再担心 `-prepareForReuse`了

#### 遵循 ASTableDelegate

在 `AnimalTableController.m` 顶部，找到以下Delegate类别接口声明：

```objc
@interface AnimalTableController (Delegate)<UITableViewDelegate>
@end
```

用 `ASTableDelegate` 替换 `UITableViewDelegate`：

```objc
@interface AnimalTableController (Delegate)<ASTableDelegate>
@end
```

现在 `AnimalTableController` 已经遵循了 `ASTableDelegate`，是时候做处理了。在 `AnimalTableController.m` 底部找到 `Delegate` 分类的实现。

我们都知道，每个 `UITableView` 至少都要提供一个 `-tableView:heightForRowAtIndexPath:` 实现方法，因为每个 cell 的高度都由代理计算和返回。

`ASTableDelegate` 中没有 `-tableView:heightForRowAtIndexPath:`。再 ASDK 中，所有的 `ASCellNode` 都负责确定自己的大小。你可以选择为单元格定义最小和最大尺寸，而不是提供静态高度。这种情况下，你希望每个cell的高度至少为屏幕的 2／3。

现在不用担心太多，这个会在第二部分中介绍。

现在只需要替换 `-tableView:heightForRowAtIndexPath:`为:

``` objc
- (ASSizeRange)tableView:(ASTableView *)tableNode 
  constrainedSizeForRowAtIndexPath:(NSIndexPath *)indexPath {
  CGFloat width = [UIScreen mainScreen].bounds.size.width;
  CGSize min = CGSizeMake(width, ([UIScreen mainScreen].bounds.size.height/3) * 2);
  CGSize max = CGSizeMake(width, INFINITY);
  return ASSizeRangeMake(min, max);
}
```

经过我们的辛勤劳动，重新编译、运行项目，看看发生了什么。

![](https://koenig-media.raywenderlich.com/uploads/2016/06/InfiniteScrollingGif.gif)

真是一个流畅的 `tableView`！一旦你开始做了，那就让我们做的更好吧！

## 无限滚动

在大多数应用中，服务器的数据点的个数往往会多于当前 tableView 中显示的单元格数量。这意味着，你必须通过某些手段做无缝处理，以便用户刷完当前数据列表时从服务端加载新的数据。

很多时候，这是通过手动观察滚动视图方法中的内容偏移来处理 `scrollViewDidScroll:`, 使用 ASDK， 有一种更具说明性的处理方式。相反的，你可以预先确定好你需要加载的页数。

你要做的第一件事是取消已经存在的方法的注释。在 `AnimalTableController.m` 的结尾，取消 `Helpers` 分类中的两个方法。你可以认为 `-retrieveNextPageWithCompletion:` 是你的网络调用，而 `-insertNewRowsInTableNode:` 是个非常典型的再表中添加新的元素的方法。

接下来，在 `-viewDidLoad` 添加：

```objc	
self.tableNode.view.leadingScreensForBatching = 1.0;  // overriding default of 2.0
```

设置 `leadingScreensForBatching` 为 **1.0** 意味着当用户滑动一个屏的时候，就会载入新的数据。

继续，在 `Delegate` 分类中实现：

```objc
- (BOOL)shouldBatchFetchForTableNode:(ASTableNode *)tableNode {
  return YES;
}
```

该方法用于告诉 `tableView` 是否继续请求新的数据。如果返回 `NO`，则在到达 API 数据末尾时，不会再不会发出任何请求。

因为你希望无限滚动，那就返回 `YES`，以确保总是请求新的数据。

接下来，还要添加：

```objc
- (void)tableNode:(ASTableNode *)tableNode willBeginBatchFetchWithContext:(ASBatchContext *)context {
  //1
  [self retrieveNextPageWithCompletion:^(NSArray *animals) {
    //2
    [self insertNewRowsInTableNode:animals];
    //3
    [context completeBatchFetching:YES];
  }];
}
```

该方法在用户滑动到 table 的末端并，且 `-shouldBatchFetchForTableNode:` 方法返回 `YES` 时被调用。

让我们回顾下上面的章节：

1. 首先，你要请求新的 animals 数据来展示。通常是通过 API 来获取的一组array。
2. 完成后，用新下载的数据更新 tableView
3. 最后，确保 `-completeBatchFetching:`返回的是`YES`，即大功告成。在完成操作之前，不会进行新的数据请求。

Build and Run，并且不停的滚呀滚。你将会看到不停的看到一只鸟，他们是无限的。

![](https://koenig-media.raywenderlich.com/uploads/2016/06/InfiniteScrollingGif.gif)

## 智能预加载

你在工作中是否曾经遇到需要预先加载内容到 scrollView 或者 pageView 控制器中？也许你正在处理一个充满屏幕 image ，并且总是希望在接下来的几张图片加载时处于等待状态，所以用户很少看到占位符。

![](https://koenig-media.raywenderlich.com/uploads/2016/03/iThinkIveGotThis-480x229.png)

当你再这样的体系下工作时，你很快就会意识到有很多问题要考虑。

- 你占用了多少内存
- 你应该提前多久加载内容
- 你决定什么时候忽略用户的交互反映

并且当你考虑到多个维度的内容时，将些问题将会变得更加复杂。假设你有一个`pageViewController`，里面每个 `viewController` 都带有一个 `collectionView`。现在，你就需要考虑如何在两个方向上动态加载内容。同时，还要对每个设备进行优化。

![](https://koenig-media.raywenderlich.com/uploads/2016/12/officespaceboss.png)

还记得告诉你 `ASRangeController` 是不重要的吗？现在，这将是我们的重点。

在每个容器类中，所有包含的 node 都有一个接口状态的概念。在任何给定的时间，一个 node 可以是下面的任意组合：

- **Preload Range（预载范围）**：通常最远的范围从可见区域。这是当cell的每个 subNode （例如ASNetworkImageNode） 的内容从外源加载，例如API和本地缓存。这与批量获取时，使用用模型对象代表cell本身形成对比。
- **Display Range（显示范围）**：在这里进行显示任务，例如文本绘制和进行图像解码。
- **Visible Range（可见范围）**：此时，node 至少有一个像素在屏幕上。

![](https://koenig-media.raywenderlich.com/uploads/2016/12/preloadingRanges-small.png)

这些范围也适用于 **screenfuls** 的度量，并且可以使用 `ASRangeTuningParameters` 属性轻松调整。

例如：你正在使用一个 `ASNetworkImageNode`在 gallery 的每个页面中展示图像，当每个cell进入 **Preload Range** 时，会发送网络请求，并且在进入 **Display Range** 时进行图像解码。

通常来说，你不必对这些 **Ranges** 太较真。利用好已有的组件，如：`ASNetworkImageNode` 和 `ASTextNode`，通常来说你将会获得极大的便利。

> **注意**: 有件不明显的事，这些 **Ranges** 不是堆栈的。相反，它们会在
**Visible Range** 上重叠和汇聚。如果将显示和预取都设置为一个屏幕，则它们将完全相同。通常数据需要存在才能显示，所以一般预取范围应该稍大一点。那么在 node 到达该范围时，就可以开始显示。

通常，该范围的前侧大于后侧。当用户改变其滚动方向时，范围的大小也是相反的，以便于对应用户实际移动的方向。

## Node接口的状态回调

你可能会疑惑：这些 **Ranges** 是如何正确工作的？很高兴你这样问~

系统中的每个 node 都有一个`interfaceState` 属性，是一个带有字段（(NS_OPTION）`ASInterfaceState`类型。`ASRangeController` 负责管理 `ASCellNode` 在 `scrolView` 上的移动，每个subNode 都由一个 `interfaceState` 属性做对应的更新。这意味着即使时 tree 中最深的 nodes 也可以相应 `interfaceState` 的变化。

幸运的是，我们很少需要直接去操作 node 的 `interfaceState` 上的 二进制位。更常见的做法时，你只需要对某 node 的特定的状态进行更改。这就是接口的状态回调。

####  Node 命名

为了看到一个 node 的各种状态，给它命名时很有必要的。这样，你就可以监测每个 node 的数据加载、内容成、屏幕展示以及所以的事情。

回到代码`-tableNode:nodeBlockForRowAtIndexPath:`,添加一句注释

	//You'll add something extra here later...
	
在它的下面，给 `cardNode` 添加一个 `debugName`：

```objc
cardNode.debugName = [NSString stringWithFormat:@"cell %zd", indexPath.row];
```
#### 观察 Cells

进入 `CardNode_InterfaceCallbacks.m` 中，你可以找到六种追踪 node 在 ranges 中的状态的方法。取消注释，Build and Run。打开你的控制台，然后慢慢滑动 table。对照你的滑动，观察cell在对应的状态变化。

> **注意**： 大多数情况下，你只要关心 `-didEnterVisibleState` 或 `-didExitVisibleState` 方法对 `ASInterfaceState` 的改变。或者说，已经为你做好了许多引擎。你可以查看 `ASNetworkImageNode` 中的代码，看看你集成的通过`Preload` 和 `Display` 状态实现的功能。 所有 node 网络图片的请求和解码，以及内存的释放都是自动完成，不费吹灰之力。

## 智能预加载（续）

在 **2.0** 版本中，已经介绍了多个维度上智能与加载的概念。假设你有一个竖直滚动的`tableView`，在其中某些Cell包含了水平滚动的 `collectionView`。

![](https://koenig-media.raywenderlich.com/uploads/2016/07/proaldGif%5E2.gif)

尽管现在的技术能够实现，但你不会希望在到达可见区域之前预先加载全部的 collection。相反的，两个方向上的 scrollView 都由各自的 `ASRangeController` 单独控制自己的 range 参数。


## 来到二次元

现在，你已经有了完整的 `AnimalTableController`， 你可以把它做为 ASPagerNode 的一个page。

项目已经提前写好了控制器的代码，首先进入 **`AppDelegate.m`**。

找到 `-installRootViewController` 的下面代码：

```objc
AnimalTableController *vc = [[AnimalTableController alloc] 
                              initWithAnimals:[RainforestCardInfo allAnimals]];
```
替换为：

```objc
AnimalPagerController *vc = [[AnimalPagerController alloc] init];
```

然后，跳到 **`AnimalPagerController.m`** 在 `-init` 方法中添加创建 `pager` 方法以及 `dataSource` 的数据源：

```objc
_pagerNode = [[ASPagerNode alloc] init];
_pagerNode.dataSource = self;
```

pagerNode 是 `ASCollectionNode` 的子类，使用方法与 `UIPageViewController` 一样。API 实际上比 `UIPageViewController` 要简单的多。

接下来要实现 pager 的 `dataSource` 方法，在底部找到 `ASPagerDataSource` 分类.

首先，告诉 pager 有几个页面。实际上当前的 animal 数组中有三组不同动物，我们需要重写 `-numberOfPagesInPagerNode:`方法：

```objc
- (NSInteger)numberOfPagesInPagerNode:(ASPagerNode *)pagerNode {
  return self.animals.count;
}
```

然后，你需要实现 `-pagerNode:nodeAtIndex` 方法，类似于先前实现的 ASTableNode 的 `dataSource` 方法。

```objc
- (ASCellNode *)pagerNode:(ASPagerNode *)pagerNode nodeAtIndex:(NSInteger)index {
    // 1
    NSArray *animals = self.animals[index];
    // 2
    ASCellNode *node = [[ASCellNode alloc] initWithViewControllerBlock:^UIViewController * _Nonnull{
        return  [[AnimalTableController alloc] initWithAnimals:animals];
    } didLoadBlock:nil];
    
    return node;
}
```

我们来总结下这部分：

1. 尽管这个版本中没有进行模块化分，但是首先获取数据模型是个好习惯。
2. 这一次，你使用的正是强大的 `-initWithViewControllerBlock:` 构造器。你所要做的就是返回一个block，这个 block 返回你提前设置好的 tableNodeController，它将自动展示在pager 的 页面中。真是太酷了😏~

一旦你添加了这个方法，你将拥有一个完整功能的 Pagar，其中的 cell 是从你原先创建的 `tableNodeController` 生成的。现在，就可以在用户的垂直和水平滑动下，充分发挥二维预加载的功能！

要查看这个 AsyncDisplayKit 2.0 教程完整的项目，[点击这里进行下载](https://koenig-media.raywenderlich.com/uploads/2016/12/AsyncDisplayKit-Finished-4.zip)。如果你想查看swift版本，[这里也有](https://koenig-media.raywenderlich.com/uploads/2016/12/RainForestSwift-1.zip)。

准备好之后，请转到该项目的第2部分，了解 AsyncDisplayKit 2.0 引入的强大的新的布局系统。

如果你想先进行深入了解，你可以阅读 [AsyncDisplayKit主页](https://asyncdisplaykit.org/) 的文档。Scott Goodson（AsyncDisplayKit的原创作者）也有几个你可能会感兴趣的话题。最近的话题很好的概述了一些框架对处理大图片存在问题的的尝试。

你可能会对 [Paper的构建](https://www.youtube.com/watch?v=OiY1cheLpmI) 感兴趣。虽然当时并没有开源，并且有许多地方发生了变化，但看到这一切的开始还是挺有意思的。

这里有一个 [public Skack channel](https://github.com/facebook/AsyncDisplayKit/issues/1582) ,欢迎来提问~

# 著作权声明

本文译自 [AsyncDisplayKit 2.0 Tutorial: Getting Started](https://www.raywenderlich.com/124311/asyncdisplaykit-2-0-tutorial-getting-started) .

由[@柏荧(BY)](http://github.com/qiubaiying)进行翻译,首次发布于 [BY Blog](http://qiubaiying.github.io)，转载请保留原文链接.