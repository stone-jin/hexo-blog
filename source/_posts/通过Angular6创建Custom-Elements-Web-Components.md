---
title: 通过Angular6创建Custom Elements/Web Components《翻译》
subtitle: 对于老项目重构难度较大、可以通过这种方式进行新功能或者局部重构成新功能
cover: https://user-images.githubusercontent.com/6525544/46247912-d975db00-c444-11e8-8e3b-ab547f79a31f.png
date: 2018-09-27 23:41:25
author: 
  nick: 金炳
  link: https://www.github.com/stone-jin
categories:
- 前端
- Angular
tags:
- Angular
---

通过新版本的Angular CLI（version 6，released 2018-04-03）和新的Angular成员（Angular Elements这个模块），让创建custom elements变得尤为简单。

如果你不知道custom elements是什么或者它跟Angular有什么关系，那大家可以查看[视频教程](https://www.youtube.com/watch?v=9zyq7FcIuvM&feature=youtu.be)来进行了解这些内容，对这些的了解有助你对文中的内容有更多的了解。

那么，不多讲，让我们来看看代码怎么写吧~

## 一、安装Angular CLI 6的版本并且初始化一个工程

```text
npm i -g @angular/cli
ng new elements-demo --prefix custom
```
我们并不需要做什么其他特殊的事情，为了web component那边，我们一般是需要以prefix前缀，此处相当于我们生成的web component元素，都是custom-{}这样的。

## 二、添加@angular/elements和polyfill
为了能使用这个功能，我们需要引入Angular的这个组件包以及一个polyfill兼容包。操作方式如下:
```text
ng add @angular/elements
```
![image](https://user-images.githubusercontent.com/6525544/46158492-8cf99680-c2b0-11e8-8956-dfb3974f4898.png)

## 三、创建一个Component
下面我们来创建一个含有Input和Output的Component，来了解它是怎么转变为能被浏览器识别的custom elements。
```text
ng g component button --inline-style --inline-template -v Native
```

最终，会帮我们在src/app中生成一个文件夹button，然后里面有个ButtonComponent。

然后我们加入一些样式和模板，最终button.component.ts看起来如下：
```typescript
import {Component, EventEmitter, Input, OnInit, Output, ViewEncapsulation} from '@angular/core';

@Component({
  selector: 'custom-button',
  template: `
    <button (click)="handleClick();">{{label}}</button>
  `,
  styles: [`
    button{
      border: solid 3px;
      padding: 8px 10px;
      background: #bada55;
      font-size: 20px;
    }
  `],
  encapsulation: ViewEncapsulation.ShadowDom // 此处我把Native改成了ShadowDom，因为Native从v6.1.0开始废弃了
})
export class ButtonComponent implements OnInit {
  @Input() label: String = 'default label';
  @Output() action: EventEmitter<number> = new EventEmitter<number>();
  private clicksCounts = 0;
  constructor() { }

  ngOnInit() {
  }

  handleClick() {
    this.action.emit(this.clicksCounts++);
  }
}

```

## 四、注册Component到NgModule
接下来就是至关重要的一步：我们通过使用Angular里面的createCustomElement这个函数来创建一个能被浏览器原生函数customElements.define使用的class。

Angular官方文档是这么描述的：
```text
createCustomElement Builds a class that encapsulates the functionality of the provided component and uses the configuration information to provide more context to the class. Takes the component factory’s inputs and outputs to convert them to the proper custom element API and add hooks to input changes.
The configuration’s injector is the initial injector set on the class, and used by default for each created instance.This behavior can be overridden with the static property to affect all newly created instances, or as a constructor argument for one-off creations.
```

然后同时，我们需要将ButtonComponent放到entryComponents里面，然后我们把原本工程自动生成的app.component.ts这些删掉。

最终我们的app.module.ts长这个样子:
```typescript
import { BrowserModule } from '@angular/platform-browser';
import {Injector, NgModule} from '@angular/core';

import { ButtonComponent } from './button/button.component';
import {createCustomElement} from '@angular/elements';

@NgModule({
  declarations: [
    ButtonComponent,
  ],
  imports: [
    BrowserModule
  ],
  providers: [],
  entryComponents: [ButtonComponent]
})
export class AppModule {
  constructor(private injector: Injector) {
    const customButton = createCustomElement(ButtonComponent, {injector});
    customElements.define('custom-button', customButton);
  }
}
```

## 五、编译、压缩并测试我们的代码
我们通过http-server来测试我们的代码，所以我们先安装下这个包
```text
npm i http-server -D
```
通常，我们使用ng build命令来编译angular的代码，然后它会帮我们生成四个文件(runtime.js, script.js, polyfills.js和main.js)，然后我们将其合并成一个js文件，为了合并方便我们让编译出来的去hash一下
```text
ng build --prod --output-hashing=none
```
最终生成到dist/elements-demo里面的，是四个不带hash的js文件。然后我们通过gzip然后变成一个js文件。
```text
cat dist/elements-demo/{runtime,polyfills,scripts,main}.js | gzip > elements.js.gz
```
然后我们在项目根目录下创建一个index.html文件。
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Title</title>
  <script src="elements.js"></script>
</head>
<body>
<custom-button label="First value"></custom-button>
<script>
  const button = document.querySelector('custom-button');
  button.addEventListener('action', (event)=>{
    console.log(`action emitted : ${event.detail}`);
  });
  setTimeout(()=>{button.label='Second Value;'}, 3000);
</script>
</body>
</html>
```
然后我们在package.json里面的scripts里面加入这几个命令：
```text
"build": "ng build --prod --output-hashing=none",
"package": "cat dist/elements-demo/{runtime,polyfills,scripts,main}.js | gzip > elements.js.gz",
"serve": "http-server --gzip"
```
然后我们通过运行 npm run build && npm run package，最后运行npm run serve.
然后我们就能看到我们web Components的页面效果了。

## 六、收尾
回顾下，操作中最关键的是什么？总结一下：
1. ng add @angular/elements库
2. 用createCustomElement创建custom elements并 customElements.define去register custom elements
3. 合并我们生成的js文件到一个压缩包

看起来并不难对吧？所以我个人也是非常激动，这是这么简单的一个过程。这将在这个使用过程中，能对原先旧项目改造的时候比较局部而简单，不需要大动干戈。

而且我们通过ls -trlh 来查看生成出来的文件的大小。elements.js.gz只有紧紧70kb，考虑到他已经涵盖了一个angular在内部，所以已经是非常可观的结果了。
