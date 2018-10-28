---
title: 5篇让你成为Angular变化检测专家《译》
date: 2018-10-28 15:00:00
subtitle: 5篇让你成为Angular变化检测专家
cover: https://cdn-images-1.medium.com/max/1600/1*Ay1gyH93CjHtaygkh5Od_w.jpeg
author:
  nick: 金炳
  link: https://www.github.com/stone-jin
categories:
  - 前端
  - Angular
tags:
  - 前端
  - Angular
---

在过去的 8 个月中，我花了大部分空闲时间对 Angular 进行逆向工程。最让我着迷的主题是变化检测。我认为它是框架中最重要的部分，因为它负责"可见"工作，如 DOM 更新，输入绑定和查询列表更新。我的探索产生了一系列深入的文章，主要突出了变量检测机制的主要思想，并深入探讨了实现细节。在这篇文章中，我将它们放在一起，并简要描述了每个内容。阅读完之后，你会获取变化检测启发。

## 理解变化检测

以下五篇深入文章，将显著提升您对 Angular 中变更检测这块的了解。每一篇文章都是基于前一篇文章中解释的信息，因此我建议你按顺序阅读他们。

### [Angular 在新版本中，脏值检测发生了新的改变](https://blog.angularindepth.com/angulars-digest-is-reborn-in-the-newer-version-of-angular-718a961ebd3e)

这篇文章对 AngularJS 的脏值检测机制和 Angular 中的更改变更机制做了对比。它解释了对他们的需求，并展示了如何使用相同的脏检查概念构建它们。然后它提供了一些示例，演示了 Angular 中的生命周期钩子如何用作 AngularJS 中$wattch 的等效机制。它还显示了 Angular 与 AngularJS 的不同之处，因为它现在强制执行所谓的从上到下的所谓单向数据流。本文解释了实施背后的原因，它对架构的好处和限制。本文对于希望迁移到 Angular，正在使用 AngularJS 的开发者很有帮助。

### [你是否依然仍未 Angular 中的变量检测需要 NgZone(zone.js)](https://blog.angularindepth.com/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular-16f7a575afef)

这篇文章描述了如何在 zone.js 库之上实现了 NgZone，并解释了 NgZone 在框架中扮演的角色。与普遍看法相反，它并不是变化检测过程的一部分，而是用于触发它。本文首先演示了 Angular 如何在没有 NgZone 和 zone.js 的情况下检测更改，并执行渲染。然后它继续展示 NgZone 带来了什么价值以及它是如何实现的。本文的大部分内容致力于解释常用的公共 API，如 isStable，onUnstable 和 onMicrotaskEmpty.本文最后解释了使用像 GoogleAPI 这样的第三方库时未检测到的变化的常见缺陷。

### [你需要知道的 Angular 中的更改检测的所有信息](https://blog.angularindepth.com/everything-you-need-to-know-about-change-detection-in-angular-8006c51d206f)

如果你想要牢固掌握变化检测机制，那么这篇文章是必读的。它提供了如何使用相关连接实现引擎的高级概述，以便进一步的探索。这篇文章先介绍了名为 View 的内部组件，并且介绍了变量检测是如何在 View 上面进行工作的。然后，它按执行顺序显示在更改检测期间执行的所有操作的列表。这些操作包括更新视图状态、渲染、处理输入绑定和调用生命周期钩子。最后，他解释了 ChangeDetectorRef 公共 API，如 detach，detectChanges 和 markForCheck，并提供了这些方法的简单示例。

### [Angular 中 DOM 更新的机制](https://blog.angularindepth.com/the-mechanics-of-dom-updates-in-angular-3b2970d5c03d)

这篇文章深入探讨了将应用程序模型与 DOM,a.k.a 单向数据绑定或 DOM 渲染呈现工程的实现细节。此操作在更改变更过程中占据了中心的位置，因为她正是在 DOM 中呈现组件更改的原因。本文首先披露了有关 View 概念的其他详细的信息，特别是 View Factory 和几种基本类型的 View 节点。然后，它显示了更改检测机制如何通过插值或输入绑定为这些节点执行 DOM 更新设置。

### [Angular 中的属性绑定更新机制](https://blog.angularindepth.com/the-mechanics-of-property-bindings-update-in-angular-39c0812bc4ce)

与前一篇关于 DOM 更新的文章类似，本文升入探讨了更新子组件和指令的输入绑定过程的实现机制。它介绍了绑定定义的概念和其在变更检测过程中的作用。然后，它继续演示了编译器在处理属性绑定的模板语法时如何生成这些绑定定义。最后，它概述了再 View 节点上运行更改检测和指令的输入属性上分布过程。

## 避免常见混淆

这里是一份附加的有价值的文章列表，主要是为了清除那些我常常在 StackOverflow 上面看到的关于变更检测方面的一些混淆。

### [认为变更检测是深度优先的人，和广度优先的人通常都是对的](https://blog.angularindepth.com/he-who-thinks-change-detection-is-depth-first-and-he-who-thinks-its-breadth-first-are-both-usually-8b6bf24a63e6)

这篇文章回答了一个有趣的问题，Angular 是否首先检查当前组件（广度优先顺序）或其子节点（深度优先）的星弟节点。它显示了 Angular 在实际开始检查他们之间如何触发兄弟组件上的生命周期钩子，并解释了这种行为如何导致您得到错误的答案。

### [你真的知道 Angular 中单向数据流的含义嘛?](https://blog.angularindepth.com/do-you-really-know-what-unidirectional-data-flow-means-in-angular-a6f55cefdc63)

这篇文章介绍了单向数据绑定和单向数据流的区别。他演示了 Angular 与 AngularJS 之间更新输入绑定的过程的不同之处，以及这种差异在何处非常的重要。

### [您需要了解有关 `Expression Changed After It Has Been Checked Error`这个错误的所有信息](https://blog.angularindepth.com/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error-e3fd9ce7dbb4)

这篇文章解释了 Angular 社区中频繁并经常被误解的错误背后的推理和机制。虽然一些开发人员将其视为一种错误，单实际上这是一个设计决策，通过将变更检测运行限制为单次运行来提高性能，而不是 AngularJS 中的大量运行($digest 运行).本文介绍了抛出错误有助于防止数据模型和 UI 之间的不一致问题。从而不会向用户展示的数据是错误的或者旧的数据。这篇文章主要由两部分构成，首先探讨了错误的原因，第二部分提出了可能的修复办法。他还解释了为什么在生产环境中为什么不会抛出这个问题。

### [如果你认为`ngDoCheck`意味着你的组件正在被检查](https://blog.angularindepth.com/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article-36ce63a3f3e5)

这篇文章为 OnPush 策略的组件触发 ngDoCheck 生命周期钩子的问题提供了详细的解答，即使这些组件的输入参数并没有发生改变。它解释了通常意外的事实，即在检查父组件时为子组件触发挂钩，并显示该机制如何触发 ngDoCheck，即使看起来没有理由这么做。本文第二部分通过演示一些例子来回答为什么我们需要 ngDoCheck 的问题。

### [Angular 中构造函数和 ngOnInit 之间的本质区别](https://blog.angularindepth.com/the-essential-difference-between-constructor-and-ngoninit-in-angular-c9930c209a42)

这篇文章中提供了一个完美的回答了最最流行的有关于 stackoverflow 上面的一个 Angular 问题，这个问题被 100K 查看，就是关于构造函数和 ngOnInit 之间的区别。本文给出了一个全面的比较，突出了使用的差异，并且还涉及了组件初始化的过程。

## Angular Air 插曲

我还强烈建议大家观看[Angular Air episode](https://www.youtube.com/watch?v=WizqXZjztss&feature=youtu.be)，在那边我谈论了视图层的内部表示和更改检测渲染的部分。我还阐述了一些与区域相关的常见误解，以及使用[ChangeDetectorRef](https://angular.io/api/core/ChangeDetectorRef)手动控制变化检测。

## 有关于 Angular 内部实现原理独一无二的书

我已经开始写一本 Angular 内部架构的综合性书籍。它将取名为<Inside Angular>。这本书深入介绍框架的架构，并且详细介绍了编译器、视图、DI、变更检测机制的工作原理。我还计划在加入一些关于性能优化和调试的例子实际例子。这本书大概 150-200 页左右，并将有大量的图表以方便大家的理解材料。如果你感兴趣，请查看您是否会购买一本关于 Angular internals 的书籍？该文章提供了一些书记的信息，并且包含了一个订阅列表，您可以用它来告诉我您是否有兴趣购买该书。
