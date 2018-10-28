---
title: Angular OnPush组件中ngDoCheck和AsyncPipe的区别
date: 2018-10-28 00:00:00
subtitle: Angular ngDoCheck和AsyncPipe在OnPush Components中的区别
cover: https://520stone-blog.oss-cn-beijing.aliyuncs.com/blog_fedfans/%E4%BF%AE%E7%BA%BF%E8%B7%AF.jpg
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

本文主要深入介绍，在 Angular 中，关于如何手动控制变动改动。

![](https://cdn-images-1.medium.com/max/1600/1*QGZXvzfMNr2LLtQqOlf5XQ.jpeg)

此文主要是为了答复 Shai 在推特上的提问。他咨询了关于用 ngDoCheck 来手动比较 values 的方法来替换使用推荐的 asyn 管道的方法 是否依然可行。这是一个比较好的提问，因为这需要对于 angular 给我们提供的 hook 有更多的了解，比如:检测机制，pipes，生命周期的 hook。接下来让我来说一下吧。

首先，我先演示如何手动触发 change detetion。这些技术使你可以更好的在 Angular 在输入绑定和异步值检查时做更好的比较。接下来让我与您分享对于这些解决方法在性能方面影响的一些看法吧。

我是一个在 ag-Grid 方面的开发拥护者。如果你对了解数据网格或者正在寻找 Angular 在数据网格这块的解决方案，可以阅读[Get started with Angular grid in 5 minutes](https://medium.com/ag-grid/get-started-with-angular-grid-in-5-minutes-83bbb14fac93)或者向我提一些问题。

接下来让我们开始吧。

## OnPush Components

在 Angular 中，我们有一个非常常见的优化技能，就是去添加 ChangeDetectionStrategy.OnPush 到 component 的 decorator 的 metadata 中。假设我们有两个简单层次的组件如下:

```
@Component({
    selector: 'a-comp',
    template: `
        <span>I am A component</span>
        <b-comp></b-comp>
    `
})
export class AComponent {}

@Component({
    selector: 'b-comp',
    template: `<span>I am B component</span>`
})
export class BComponent {}
```

通过这样的配置，Angular 每次都会对 A 和 B 组件始终运行更改检测。如果现在我们给 B 组件添加了 OnPush 的策略的话。

```
@Component({
    selector: 'b-comp',
    template: `<span>I am B component</span>`,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {}
```

只有在其输入绑定发生更改的时，Angular 才会对 B 组件运行更改检测。上面由于此时它并没有进行输入绑定，所以在启动期间，只会对 B 组件进行一次更改检测。

## 手动触发变更检测

那么是否有一种给 B 组件强制触发更改检测的方法呢？答案，当然是的，我们可以通过注入 changeDetectorRef，并且使用 markForCheck 方法来告知 Angular 当前组件需要进行更改检测。因为根据生命周期，NgDoCheck hook 会被触发，所以我们可以用下面的方法:

```
@Component({
    selector: 'b-comp',
    template: `<span>I am B component</span>`,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {
    constructor(private cd: ChangeDetectorRef) {}

    ngDoCheck() {
        this.cd.markForCheck();
    }
}
```

然后，当 Angular 检查父组件 A 的时候，会对 B 进行变更检测。接下来让我们看看如何使用吧。

## 输入绑定

我们说过，Angular 只会在绑定发生变化的时候，对 OnPush 的组件进行变更检测。所以让我们看一个输入绑定的例子。假设我们有一个通过从父组件传递下来的输入对象：

```
@Component({
    selector: 'b-comp',
    template: `
        <span>I am B component</span>
        <span>User name: {{user.name}}</span>
    `,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {
    @Input() user;
}
```

在父组件 A 中，我们定义了一个对象，并且实现了 changeName 的方法，这个方法需要在单击按钮时，更新这个对象的名字：

```
@Component({
    selector: 'a-comp',
    template: `
        <span>I am A component</span>
        <button (click)="changeName()">Trigger change detection</button>
        <b-comp [user]="user"></b-comp>
    `
})
export class AComponent {
    user = {name: 'A'};

    changeName() {
        this.user.name = 'B';
    }
}
```

如果现在我们运行这个例子，则再第一次变更检测之后，我们将会看到用户的 name 打印:

```
User name: A
```

但是当我们点击了按钮，并且在回调函数中改变了变量的名字:

```
changeName() {
    this.user.name = 'B';
}
```

名字并不会在屏幕上更新。我们知道这是因为 Angular 对 Input 的参数只进行浅比较，此处 user 变量的引用没有发生变化。那么我们怎么来解决这个问题呢?

好吧，我们可以在检测到差异时，手动检查名称并触发变化检测:

```
@Component({
    selector: 'b-comp',
    template: `
        <span>I am B component</span>
        <span>User name: {{user.name}}</span>
    `,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {
    @Input() user;
    previousName = '';

    constructor(private cd: ChangeDetectorRef) {}

    ngDoCheck() {
        if (this.previousName !== this.user.name) {
            this.previousName = this.user.name;
            this.cd.markForCheck();
        }
    }
}
```

如果你运行了这个代码，你将在屏幕上看到更新的名字。

## 异步更新

现在，让我们的例子更复杂一些。我们将介绍一种基于 RxJs 的服务，它可以异步发出一个更新。它有点类似于 NgRx 体系结构。我将使用 BehaviorSubject 作为值的来源，因为我需要以初始值启动流:

```
@Component({
    selector: 'a-comp',
    template: `
        <span>I am A component</span>
        <button (click)="changeName()">Trigger change detection</button>
        <b-comp [user]="user"></b-comp>
    `
})
export class AComponent {
    stream = new BehaviorSubject({name: 'A'});
    user = this.stream.asObservable();

    changeName() {
        this.stream.next({name: 'B'});
    }
}
```

因此，我们在子组件中收到此用户变量流。我们需要订阅流并检查值是否更新。这样做的常用方法是使用异步管道。

## 异步管道

所以这里是子组件 B 的实现：

```
@Component({
    selector: 'b-comp',
    template: `
        <span>I am B component</span>
        <span>User name: {{(user | async).name}}</span>
    `,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {
    @Input() user;
}
```

[查看例子](https://stackblitz.com/edit/angular-q8n3qj?file=src%2Fapp%2Fa.component.ts)。但是有另一种不使用异步管道的方法吗？

## 手动检查和更改检测

是的，我们可以手动检查值并在需要的时候来触发更改检测。正如我们开头的例子一样，我们可以使用 NgDoCheck 的生命周期的钩子:

```
@Component({
    selector: 'b-comp',
    template: `
        <span>I am B component</span>
        <span>User name: {{user.name}}</span>
    `,
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {
    @Input('user') user$;
    user;
    previousName = '';

    constructor(private cd: ChangeDetectorRef) {}

    ngOnInit() {
        this.user$.subscribe((user) => {
            this.user = user;
        })
    }

    ngDoCheck() {
        if (this.previousName !== this.user.name) {
            this.previousName = this.user.name;
            this.cd.markForCheck();
        }
    }
}
```

我们可以体验下: [例子](https://stackblitz.com/edit/angular-4xuug1?file=src%2Fapp%2Fb.component.ts)

理想情况，我们希望从 NgDoCheck 移动我们的比较和更新逻辑并将其放到订阅回调中，因为那时新值将可用:

```
export class BComponent {
    @Input('user') user$;
    user = {name: null};

    constructor(private cd: ChangeDetectorRef) {}

    ngOnInit() {
        this.user$.subscribe((user) => {
            if (this.user.name !== user.name) {
                this.cd.markForCheck();
                this.user = user;
            }
        })
    }
}
```

[例子](https://stackblitz.com/edit/angular-lvtfve?file=src%2Fapp%2Fb.component.ts)

有趣的是，这正是异步管道在幕后做的事情。

```
@Pipe({name: 'async', pure: false})
export class AsyncPipe implements OnDestroy, PipeTransform {
  constructor(private _ref: ChangeDetectorRef) {}

  transform(obj: ...): any {
    ...
    this._subscribe(obj);

    ...
    if (this._latestValue === this._latestReturnedValue) {
      return this._latestReturnedValue;
    }

    this._latestReturnedValue = this._latestValue;
    return WrappedValue.wrap(this._latestValue);
  }

  private _subscribe(obj): void {
    ...
    this._strategy.createSubscription(
        obj, (value: Object) => this._updateLatestValue(obj, value));
  }

  private _updateLatestValue(async: any, value: Object): void {
    if (async === this._obj) {
      this._latestValue = value;
      this._ref.markForCheck();
    }
  }
}
```

## 那么哪一种解决方案更快?

现在我们知道了如何使用手动更改检测而不是异步管道，让我们回答我们最开始的问题。谁更快？

嗯，这取决于你如何比较他们，但在其他条件相同的情况下，手动方法会更快。我不认为这种区别是有形的。以下是为什么手动方法可以更快的几个例子。

就内存而言，您不需要创建 Pipe 类的实例。就编译而言，编译器不必花时间解析管道特定语法并生成管道特定输出。在运行时方面，您可以使用异步管道为组件上的每一个更改检测运行保存几个函数调用。这是为管道代码生成的 updateRenderer 函数的方法:

```
function (_ck, _v) {
    var _co = _v.component;
    var currVal_0 = jit_unwrapValue_7(_v, 3, 0, asyncpipe.transform(_co.user)).name;
    _ck(_v, 3, 0, currVal_0);
}
```

正如您所见，异步管道的代码调用管道实例上的 transorm 方法以获取新值。管道将返回从订阅中收到的最新值。

将其与手动方法生成的普通方法进行比较：

```
function(_ck,_v) {
    var _co = _v.component;
    var currVal_0 = _co.user.name;
    _ck(_v,3,0,currVal_0);
}
```

这些是 Angular 在检查 B 组件时执行的功能。

## 一些更有趣的事情

与执行浅比较的输入绑定不同，异步管道的实现变更不执行比较（感谢 Olena Horal 特别提到的）。它将每个新的 emission 作为更新处理，即使它与之前的 emission 相同。这是发成相同对象父组件 A 的实现。尽管如此，Angular 仍然运行 B 组件的变化检测:

```
export class AComponent {
    o = {name: 'A'};
    user = new BehaviorSubject(this.o);

    changeName() {
        this.user.next(this.o);
    }
}
```

这意味着每次发出新值时，都会标记具有异步管道的组件以进行检测。并且 Angular 将在下次运行变更检测时检查组件，即使该值未更改。

这有什么关系？好吧，在我们例子中，我们只对用户的属性名称该兴趣，因为我们在模板中使用它。我们并不关心整个对象以及对对象应用可能会改变的事实。如果名称相同，我们不需要重新渲染组件。但你无法用异步管道来避免这种情况。

NgDoCheck 本身并非没有问题:)由于只有在检查了父组件时才会触发钩子，如果其中一个父组件使用了 OnPush 策略并且在更改变更期间未做检查，则不会触发该钩子。因此，当您通过服务收到新值时，不能依赖它来触发更改检测。在这种情况下，我在订阅回调中使用 markForCheck 显示的解决方案是可行的方法。

## 结论

基本上，手动比较可以让您更好的控制检查。您可以定义何时检查组件。这与很多其他工具相同-手动控制为您提供了更大的灵活性，但您必须知道自己在做什么。为了获得这些知识，我鼓励您投入时间和精力来学习和阅读资源。

如果您担心调用 NgDoCheck 生命周期的评率，或者它会比管道变更更换更频繁的调用。首先请不要做。首先我们展示了上面的解决方案，您不使用异步流的手动方法中的钩子。其次，只有在检查父组件时才会调用钩子。如果未选中父组件，则不会调用该挂钩。关于管道，由于流中的浅层检查和更改引用，您将使用管道的转换方法火的相同数量的调用或甚至更多。

## 想要了解更多有关 Angular 中变更检查的更多信息？

首先，阅读[These 5 articles will make you an Angular Change Detection expert.](https://www.520stone.com/page/article/五篇让你成为Angular变化检测专家/).如果你想要牢固掌握 Angular 中的变化检测机制，那么这个系列是必读的。每篇文章都以前一篇文章中解释的信息为基础，然后以高级的维度讲解实现细节。
