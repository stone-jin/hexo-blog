---
title: Angular在新版本中，脏值检测重生了《译》
date: 2018-10-28 16:15:00
subtitle: Angular在新版本中，脏值检测重生了
cover: https://cdn-images-1.medium.com/max/800/1*DEeCWAZ_UhN36xf4aY4EBg.jpeg
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

Angular's $digest is gone. Long live the digest!

我用 Angular.js 工作了几年，尽管受到了广泛的批评，但我依然认为这是一个很棒的框架。我从<Builing your own Angular.js>这本书入手，并且阅读了大部分框架的源码。所以我对 Angular.js 内部工作有了扎实的了解，并且很好的掌握了框架构建的思想。现在，我试图在更新后的 Angular 中 达到相同的理解水平，并且在版本之间映射想法。我发现，与互联网声称的 Angular 相反，Angular 还是借用了其前身的很多想法。

其中一个想法就是臭名鼎鼎的[循环脏值检测](https://larseidnes.com/2014/11/05/angularjs-the-bad-parts/):

这个操作在 Angular.js 中非常昂贵。改变应用程序的任何一部分将成为数百或者上千个查找更改的函数的操作。这是 Angular.js 的基本组成部分，它对可以在 Angular 中构建的 UI 的大小设置了一个硬性限制，同时保持高性能。

如果对 Angular 的脏值检测实现机制有了很好的理解，那我们同样可以将应用程序设计地非常的高效。例如：有选择性地使用$scope.$digest()而不是$scope.$apply，而不是所有地方都是用$apply，拥抱不可变对象。但事实上，需要对底下实现有一定了解才能设计高性能的应用。

因此，大多数关于 Angular 的教程都不奇怪框架中没有更多的$digest 循环。这种观点很大程度取决于我们对脏值检测的理解，但是我认为，鉴于其目的，这是一种误导性的主张。它还在那里。是的，我们没有明确的范围和观察者，也没有调用$scope.$digest，但是检查遍历组件数的更改机制，调用隐式观察者炳更新 DOM 节点。最终完全被改写，并且被大大加强了。

这篇文章探讨了 Angular.js 和 Angular 在检测这块的实现的差异。并且对于 Angular.js 的开发者会有帮助，在他们迁移到 Angular 中。

## 变更的需要

在我们开始之前，让我们记住为什么脏值检测会在 Angular.js 中出现。每个框架解决了数据模型和 UI 之间的同步问题。这个实事过程中最大的挑战就是更改检测。这也是当今大部分知名框架在实现之间最大的区别了。我打算写一篇深入介绍变更检测机制比较的文章。如果你想要被提醒到，请关注我谢谢:)

检查变更的方法有两种主要方法--通过用户告知框架或者自动探测更改。假设我们有下面这样一个对象:

```
let person = {name: 'Angular'};
```

然后我们更新了 name 字段。我们的框架如何知道它发生了变化呢？一种方法是让用户来通知框架。

```
constructor() {
    let person = {name: 'Angular'};
    this.state = person;
}
...
// explicitly notifying React about the changes
// and specifying what is about to change
this.setState({name: 'Changed'});
```

或者强迫他在属性上使用一个包装器，以方便框架添加 setter:

```
let app = new Vue({
    data: {
        name: 'Hello Vue!'
    }
});
// the setter is triggered so Vue knows what changed
app.name = 'Changed';
```

另一种方法是保存一个 name 属性的前一个值，并将其与当前值进行比较.

```
if (previousValue !== person.name) // change detected, update DOM
```

但什么时候应该被比较呢？我们应该在每次代码运行的时候运行检查机制。而且我们知道代码是异步时事件运行的-所谓的虚拟机 VM(反向、勾选)，我们可以在检测结束的时候进行检查操作。这就是 Angular.js 使用脏值检查的原因。所以我们可以将变更检测定义为

```
一种更改检测机制，用于遍历组件树，检查每个组件的变化，并在组件属性变化发生Dom的更新
```

如果我们使用了这个变更检测的定义，我断言主要机制并没有在新版本的 Angular 中改变。更改的知识实现变更检测的实现。

## Angular.js

Angular.js 使用了观察者和监听器的概念。一个观察者是一个用来返回一个被监听的对象的值的函数。通常，这些值是一个数据模型上的属性。但它并不总是数据模型上面的属性--我们也可以跟踪 scope 的状态，计算值，一个第三方组件。如果监听的值相比前面的值不一样了，angular 就会调用监听器。这个监听器通常是用来更新 UI 的。

这反应在$watch 函数的参数中：

```
$watch(watcher, listener);
```

所以，如果我们有一个 person 这样的对象，这个对象里面有一个用于 html 里面展示的 name 字段，如下:

```
<span>{{name}}</span>
我们可以通过下面的方法跟踪这个属性并更新DOM
```

$watch(() => {
return person.name
}, (value) => {
span.textContent = value
});

```
这基本上就像ng-bind这样的插值和指令。Angular.js使用指令来反射数据在DOM上的表现。最新的Angular不在那么做了。它使用了一个属性映射表来连接数据模型和DOM。现在的实现方式如下:
```

<span [textContent]="person.name"></span>

```
由于我们有许多构成树的组件，并且每一个组件都有不同的数据模型，因此我们有一个跟组件树非常相似结构的观察者层次结构。观察者使用$scope进行分组，但这并不重要。

现在，在angular.js变更检测在这个观察者树结构中走过，并且更新DOM。通常，如果你使用现有的机制$time，$http，或者通过$scope.$apply或者$scope.$digest会触发一个异步事件。

监听器会按照严格的顺序进行处罚，首先是父组件，然后是子组件。这是有道理的，但它有一些不受欢迎的含义。观察者监听器可以具有各种副作用，包括更新父组件的属性。如果已经处理了父组件，然后一个子组件更新了父组件的属性，则不会检测到更改。这就是为什么更改检测需要必须多次运行才能保持稳定-以便不再有更改。并且此类运行的数量限制是10.这个涉及限制被认为是有缺陷的，Angular不允许这么做。

## Angular

Angular没有类似于Angular.js的观察者的概念。但是跟踪数据模型属性变化的函数还是存在。这些变更函数现在是由框架编译器生成，无法访问。此外，它们现在与底层DOM紧密项链。这些函数被存储在视图View上的[updateRenderer](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/view.ts#L140)的属性名称中。

它们也非常的具体--它们只跟踪数据模型中的变化，而不像Angular.js中跟踪所有的内容。每一个组件由一个观察者，它跟踪模板中使用的所有组件属性。它不是返回一个值，而是返回的是每一个被跟踪属性调用[checkAndUpdateTextInline](https://github.com/angular/angular/blob/6b79ab5abec8b5a4b43d563ce65f032990b3e3bc/packages/core/src/view/text.ts#L62)的函数。这个函数会对前面的值与当前值做对比，然后当改变的时候对DOM进行更改。

例如，对于AppComponent有如下一个模板：
```

<h1>Hello {{model.name}}</h1>
```

编译器会转换成下面的代码:

```
function View_AppComponent_0(l) {
    // jit_viewDef2 is `viewDef` constructor
    return jit_viewDef2(0,
        // array of nodes generated from the template
        // first node for `h1` element
        // second node is textNode for `Hello {{model.name}}`
        [
            jit_elementDef3(...),
            jit_textDef4(...)
        ],
        ...
        // updateRenderer function similar to a watcher
        function (ck, v) {
            var co = v.component;
            // gets current value for the component `name` property
            var currVal_0 = co.model.name;
            // calls CheckAndUpdateNode function passing
            // currentView and node index (1) which uses
            // interpolated `currVal_0` value
            ck(v, 1, 0, currVal_0);
        });
}
```

因此，即使现在以不同的方式实现了观察者，变更检测的循环还是存在。它更改了名称以更改检测周期。

```
在开发阶段，tick()会执行两次change detection cycle来确保没有新的改变被探测到。
```

我之前提到过，在 angular.js 的变更探测会走一遍观察者树，并且更新 DOM。Angular 也做了非常相像的事情。当更改周期变动的时候，会走一遍组件树，并且调用渲染更新的函数。它是作为检查和更新视图过程的一部分完成的，我在关于 Angular 变化检测需要了解的所有内容中介绍了他。

就像 Angular.js 在教新版本中的 Angular，这个变更检测周期会被每一个异步事件触发。但是，由于 Angular 使用了 zone 来修复所有的异步时间，因此大部分事件不需要手动被触发。框架订阅了 onMicrotaskEmpty 时间，并在异步事件完成的时候收到通知。当 VMturn 中没有排队的微任务的时候就会触发此事件。但是，可以通过 view.detectChanges 或 ApplicationRef.tick 方法来手动触发更改检测。

Angular 强制使用了所谓的从上到下的单向数据流模型。在处理父组件被处理完毕后，不允许层次结构较低的结构更新父组件的属性。如果组件在 DoCheck 挂钩中更新了父组件模型属性，则可以正常工作，因为在检测到属性更改之前会调用此声明周期挂钩。但是，如果以其他方式去更新父组件的属性，例如，在处理更改后调用的 AfterViewChecked 挂钩是，则会在开发模式下获得以下报错:

```
Expression has changed after it was checked
```

你可以在文章《[您需要了解有关 `Expression Changed After It Has Been Checked Error`这个错误的所有信息](https://blog.angularindepth.com/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error-e3fd9ce7dbb4)》了解更多的信息。

在生产环境中，这个错误将不会出现，只有当 Angular 执行下一个更改周期的时候才会探测到这个改动。

## 使用生命周期钩子来跟踪变化

在 Angular.js 中，每一个组件定义了一系列的跟踪者来跟踪下面这些信息:

- 父组件的数据绑定
- 自己组件的属性
- 计算的值 ( computed value)
- Angular 生态圈以外的第三方组件

以下是如何在 Angular 中实现这些功能的。要跟踪父组件绑定属性，我们会使用 OnChange 生命周期的钩子。

我们可以使用 DoCheck 生命周期来跟踪自己组件属性和计算属性。因为这个周期会在 Angular 流程属性发生更改之前触发，因此我们可以执行任何我们操作以反应到界面上。

我们可以使用 OnInit 来监听 Angular 生态圈以外的变化，并且手动探测变化。

例如，我们有一个展示当前时间的组件。这个时间由 Time Servicce 提供。这是 Angular.js 中的实现方式。

```
function link(scope, element) {
    scope.$watch(() => {
        return Time.getCurrentTime();
    }, (value) => {
        $scope.time = value;
    })
}
```

下面是我们在 Angular 中的实现方式：

```
class TimeComponent {
    ngDoCheck()
    {
        this.time = Time.getCurrentTime();
    }
}
```

另一个例子，如果我们有一个第三方的滑块的组件，没有集成到 Angular 的生态系统中，单我们需要展示当前的页面，我们只需要简单的包装秤一个 Angular 组件，来跟踪滑块的更改时间，并且手动触发更改来反映到 UI 中。

```
function link(scope, element) {
    slider.on('changed', (slide) => {
        scope.slide = slide;

        // detect changes on the current component
        $scope.$digest();

        // or run change detection for the all app
        $rootScope.$digest();
    })
}
```

Angular 的想法也是一样的。以下是它的实现方式:

```
class SliderComponent {
    ngOnInit() {
        slider.on('changed', (slide) => {
            this.slide = slide

            // detect changes on the current component
            // this.cd is an injected ChangeDetector instance
            this.cd.detectChanges();

            // or run change detection for the all app
            // this.appRef is an ApplicationRef instance
            this.appRef.tick();
        })
    }
}
```

这就是全部的内容了!
