
---
title: Angular7.0发布-虚拟滚动、拖拽等特性
date: 2018-10-19 07:38:06
subtitle: Angular7.0带来更多新特性
cover: http://520stone-blog.oss-cn-beijing.aliyuncs.com/blog_fedfans/1920X1080.jpg
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

2018年10月19日，Angular7.0发布了! 这是一个跨了major版本的版本，其中包括核心框架，Angular Material，还有Angular CLI跟主分支也进行了同步。这个版本在工具链方面提供了更多的特性，并且合并了一些主要参与者的分支的功能。

![](https://cdn-images-1.medium.com/max/1600/1*CQKUmJrBs-523I4GOiEUaA.gif)
虚拟scrolling可以提高我们的应用性能

# 如何升级到Angular7.0的版本
还是跟以往一样，我们可以通过 [https://update.angular.io](https://update.angular.io)查看详细的文档和手册来升级我们的项目。同时告诉大家一个好消息，就是因为我们在V6.0的版本中做的工作，我们大部分的开发者可以通过一条命令就能从Angular6.0升级到Angular7.0版本。

```
ng update @angular/cli @angular/core
```

早期已经体验过这个升级过程的开发说，这次升级相比以往变得更快了，大部分升级过程仅仅花了不到10分钟就升级完了。

# CLI提示输入
现在Angular/cli工具会在当用户运行ng new 或者ng add @angular/material的时候，提示用户一些参数信息，来帮助用户使用routing或者scss功能的功能。

这个提示用户帮助创建项目的功能，已经被加入到@angular-devkit/schematics-cli这个工具中，关于schematics，又可以跳转到另一篇[文章](https://blog.angular.io/schematics-an-introduction-dc1dfbc2a2b2),这样任何用schematics发布的用户可以通过添加x-promt到Schematics的配置项中。

如下schematic.json文件:
```
"routing": {
  "type": "boolean",
  "description": "Generates a routing module.",
  "default": false,
  "x-prompt": "Would you like to add Angular routing?"
},
```

# 应用性能方面
还是跟以往一样，我们依旧关注着性能方面的改进，我们分析了大部分Angular生态圈这边的错误。我们发现有很多的开发者在生产环境中加入了reflect-metadata这个polyfill，这其实仅仅只需要在开发阶段加入即可。

为了解决这个问题，升级到Angular7.0的过程中，我们会自动从polyfills.ts文件中移除reflect-metadata这个，然后当我们使用JIT模式的编译的时候加入这个reflect-metadata，而在生产环境编译中默认移除这个。

在Angular7.0的版本中，我们会提醒新项目使用cli工具打包的时候，生成的包的大小。新应用，我们会在2MB大小的时候进行警告，而在5MB的时候直接报错。当然这个大小我们增加了配置项在angular.json文件中让我们进行配置。

```
"budgets": [{
  "type": "initial",
  "maximumWarning": "2mb",
  "maximumError": "5mb"
}]
```

关于包的提示的内容，我们会在Chrome上面进行显示。如下图:

![](https://cdn-images-1.medium.com/max/1600/1*jXHBMok5cNnkXD8O0a8gAg.png)

# Angular Material & the CDK
Material 设计组在2018年做了一个比较大的改动，具体改动，我们可以查看文档[a big update](https://www.youtube.com/watch?v=1Dh8ZBQp9jo)。Angular Material的用户在升级到Angular7.0的版本中，我们需要关注下在表现形式上面的一些微小的展示上的不同。
如下图：
![](https://cdn-images-1.medium.com/max/1600/1*lgZYt3RBGM_c7HUcg85Zgg.png)

在angular material的CDK中，我们又利用了虚拟滚动和拖拽的功能，加入了DragDropModule或者叫Scrolling Module。

## 虚拟滚动
虚拟滚动让DOM节点在一个list中进行加入和删除，让我们的在一个大型可滚动的应用中体验到更加流程的用户体验。

代码:
```
<cdk-virtual-scroll-viewport itemSize="50" class="example-viewport">
  <div *cdkVirtualFor="let item of items" class="example-item">{{item}}</div>
</cdk-virtual-scroll-viewport>
```

如果想要了解更多关于 Virual Scrolling，可以查看[文章](https://material.angular.io/cdk/scrolling/overview)

## 拖拽

如下图的拖拽效果:

![](https://cdn-images-1.medium.com/max/1600/1*i30ZQdBC7CKbXXdOrUNQcg.gif)

在CDK中，我们加入了拖拽的功能，当用户移动的时候我们会自动的重绘元素就，这里又一些方法可以提供给我们，moveItemInArray，还有在列表间传送单元:transferArrayItem。

```
<div cdkDropList class="list" (cdkDropListDropped)="drop($event)">
  <div class="box" *ngFor="let movie of movies" cdkDrag>{{movie}}</div>
</div>
```

```
  drop(event: CdkDragDrop<string[]>) {
    moveItemInArray(this.movies, event.previousIndex, event.currentIndex);
  }
```

如果想要了解更多关于拖拽这块的，可以查看[文章](https://material.angular.io/cdk/drag-drop/overview)

# 同时我们优化了selects的可用性
我们通过在mat-form-field这个标签中使用原生select节点之中，提高了应用的可用性。原生的select表现出更好的性能，可用性和可获得性，但是我们也将继续保留mat-select做为mat-form-fields的内置标签使用。
(这个也是在Angular Material中的)

如果想要了解mat-select和mat-form-field的内容，我们可以查看[链接](https://material.angular.io/components/select/overview)

# Angular Elements
Angular Elements现在通过标准的custom elements，支持了内容的保护。
如下:
<my-custom-element>This content can be projected!</my-custom-element>

关于Angular Elements，可以看我的翻译的文章[通过Angular6创建Custom Elements/Web Components](https://www.520stone.com/page/article/%E9%80%9A%E8%BF%87Angular6%E5%88%9B%E5%BB%BACustom-Elements-Web-Components/)

# Angular项目参与者提供的功能

Angular在社区这块非常感激大家的贡献，同时我们在一些社区项目中参与了，并且一些项目已经正式使用了。如下图:
StackBlitz 2.0 Supports multipane editing and the Angular Language Service
![](https://cdn-images-1.medium.com/max/1600/1*5dDsNd840QO7btuZUsnBYw.gif)

[Angular Console](https://angularconsole.com/)一个可以被下载的console，用来做项目启动和运行的。
[@angular/fire](https://github.com/angular/angularfire2) AngularFIle有了一个新的报名，他有当前最稳定的版本
[NativeScript](https://docs.nativescript.org/code-sharing/intro)通过NativeScript，让我们既能让我们项目运行成web，也能运行成可安装的手机应用。
[StackBlitz](https://stackblitz.com/fork/angular)StackBlitz2.0已经发布

# 升级手册
我们在不断优化我们的使用手册。现在官方的angular.io中，已经包含了angular-cli的文档，文档地址:[地址](https://angular.io/cli)

# 依赖升级
Angular7.0依赖于Typescript3.1, RxJS6.3，Node10（我们也会支持Node8）

# Ivy怎么样了？
我们还在努力为我们下一代的渲染引擎，Ivy努力。Ivy正在开发中，但还不是v7里面的功能。我们正在验证在一些过去应用中的可用性，并且最快的预计下个月，我们会给出Ivy的预览效果。
