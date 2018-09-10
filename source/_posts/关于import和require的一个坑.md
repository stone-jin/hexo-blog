---
title: Node.js里面import和require一样吗？
date: 2018-09-10 20:28:31
subtitle: import * as http from 'http'; 和 const http = require('http')这两个是否是相同的？
cover: http://520stone-blog.oss-cn-beijing.aliyuncs.com/blog_fedfans/1920X1080.jpg
author: 
  nick: 金炳
  link: https://www.github.com/stone-jin
categories:
- 后端
- Node
tags:
- 问题
- NodeJS
---

## 内容简介
本文主要讲解内容：import * as http from 'http'; 和 const http = require('http')这两种写法是否效果是相同的？

## 问题场景篇
我的ts的代码里面，把import * as http from 'http'; 和 const http = require('http')，这两个表现不一样。

如下代码: A.ts是我封装的一个包，相当于内部，我使用shimmer对这个http的createServer这个函数进行hook，然后方便做代码http接口调用情况的统计。main.ts假设是业务系统里面的代码，内部假设使用的是var koa = require('koa')，底层肯定是http的createServer，所以我们此处用var http = require('http')来模拟，然后发现A.ts包里面如果我用import * as http from 'http';则hook不了对应的createServer的代码，但是我用const http = require('http')就可以hook。

A.ts
```typescript
import * as http from 'http';
import * as shimmer from 'shimmer';

shimmer.wrap(http, 'createServer', original=>{
   return function(this: any, requestListener: any){
       console.log("====>")
       console.log("2222")
       // 此处hook最底层的http,createServer
       // @ts-ignore
       const result = original.apply(this, arguments);
       return result;
   }
});
```

main.ts(业务代码里面，假设是require('http'))
```typescript
var http = require('http');

const proxy = http.createServer((req: any, res : any) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('okay');
});
```

## 问题排查
然后将对应的ts代码编译代码后发现: import 的代码编译成了下面这种代码
```javascript
var __importStar = (this && this.__importStar) || function (mod) {
    if (mod && mod.__esModule) return mod;
    var result = {};
    if (mod != null) for (var k in mod) if (Object.hasOwnProperty.call(mod, k)) result[k] = mod[k];
    result["default"] = mod;
    return result;
};
const http = __importStar(require("http"));
```
相当于我写的代码为什么不是
```javascript
const http = require('http')
```
呢？不然我hook的其实是这个__importStar里面这个result上面的createServer这个函数

最终，查到原来是ts的配置项中的esModuleInterop被设置成了true导致的。我们查看tsconfig.json文件里面的描述
```text
Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'
```
相当于是为了做一个CommonJS和es modules这边的一个互通，会帮忙新建一个namespace objects。所以相当于这个__importStar是为了跟es modules那边有关系。

解决办法:把tsconfig.json里面这个特性关掉
```text
esModuleInterop: false
```

## 继续深入查询
然后我们看一下babel那块对于import这块的处理，我们写一个main.js
```javascript
import * as http from 'http';

http.get("http://www.baidu.com");
```
然后项目安装babel的依赖
```text
npm install --save-dev babel-preset-es2015

# ES7不同阶段语法提案的转码规则（共有4个阶段），选装一个
$ npm install --save-dev babel-preset-stage-0
$ npm install --save-dev babel-preset-stage-1
$ npm install --save-dev babel-preset-stage-2
$ npm install --save-dev babel-preset-stage-3
```

然后新建一个文件.babelrc文件
```text
  {
    "presets": [
      "es2015",
      "stage-2"
    ]
  }
```

然后我们通过调用babel main.js --out-file out.js,查看过babel之后的这个包

```javascript
"use strict";

var _http = require("http");

var http = _interopRequireWildcard(_http);

function _interopRequireWildcard(obj) { if (obj && obj.__esModule) { return obj; } else { var newObj = {}; if (obj != null) { for (var key in obj) { if (Object.prototype.hasOwnProperty.call(obj, key)) newObj[key] = obj[key]; } } newObj.default = obj; return newObj; } }

http.get("http://www.baidu.com");
```
然后我们也看到了一个类似__importStar这样的函数，也就是_interopRequireWildcard，也是内部去创建了一个{}。

## 对比
所以我们最终可以得到一个结果，babel这边
```javascript
import * as http from 'http';
```
其实跟const http = require('http')是有区别的，babel这边会帮我们生成一个类似namespace这样的东西。相当于这个文件里面里面http是处于一个namespace里面的，两个文件都import * as http from 'http'; 如果其中一个文件修改了，则另外一个是不会被修改的。

然后typescript编译器那边，给我们加了这样一个属性，就是是否需要这种特性，也就是配置项: esModuleInterop

## 总结
那对于我的需求来说，我是为了让调用这个npm方的人，对于大家调用的底层的http包进行hook，所以我不希望有这样一个namespace包起来，所以我的ts代码里面就需要把这个esModuleInterop给关闭掉。最终编译后的代码:

```javascript
const http = require("http");
//简称这个地方不会帮我们包一层namespace
```

所以Node.js里面，typescript，关于模块引用，我们可以选择用es module的形式去引用一个包，也可以用commonJS的方式去引用一个包，如果你是用es6在写。那es6那边你import了一个包，那就会多一个namespace这样的东西。如果有不对，请帮忙指正下。