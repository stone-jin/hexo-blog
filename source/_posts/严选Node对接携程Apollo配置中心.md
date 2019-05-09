---
title: 严选Node对接携程Apollo配置中心
cover: https://cdn.nlark.com/yuque/0/2019/png/187105/1557386231926-fb7107ea-fbea-41fd-8e42-c334fbc68547.png
subtitle: 严选的配置中心是基于Apollo配置中心开发的，为了跟Java共用一个配置中心，我们用Node对接了携程Apollo配置中心，希望能给大家带来一些帮助。
date: 2019-5-9 22:13:18
author: 
  nick: 金炳
  link: https://www.github.com/stone-jin
categories:
- Node
tags:
- Node
- 配置中心
- Apollo
- 基建
---

# 严选Node对接携程Apollo配置中心

## 目录
![](https://cdn.nlark.com/yuque/0/2019/png/187105/1557414849981-a5f61a71-4720-4f78-8801-e1e7d6068a5e.png)

<a name="s38lq"></a>
## 一、背景
<a name="TF7Fh"></a>
### 1.1 对接携程配置中心
因为在给Node支持携程Apollo对接，携程Apollo是一个配置中心的解决方案，配置中心又是服务端的一个比较常用的功能。网上有人开发了一个node的apollo的sdk：[https://github.com/Quinton/node-apollo](https://github.com/Quinton/node-apollo)，查看代码和介绍发现并没有实现配置动态变化的时候，如何下发到应用内，其内部是主要是拉取config配置和生成对应的配置，相当于如何将远程的配置拉过来生成一份本地的配置。

<a name="OPQsV"></a>
### 1.2 需求
那么我们的需求是：能跟java的客户端一样，当应用起来的时候，我们能从配置中心拉取当前新的配置文件，如果跟配置中心网络是断开的，则我们先使用本地的配置文件。如果连接上后，当配置中心上，我们更新了新的配置文件的话，需要实时同步到应用程序内部确保应用程序用的是新的配置文件。

<a name="XrMJN"></a>
### 1.3 官方客户端设计思路
官方总体对于 客户端设计思想如下图：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557386231926-fb7107ea-fbea-41fd-8e42-c334fbc68547.png#align=left&display=inline&height=738&name=image.png&originHeight=738&originWidth=1520&size=132695&status=done&width=1520)

上图也就跟我们的需求很一样，官方描述一下Apollo客户端的实现原理：

1. 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
1. 客户端还会定时从Apollo配置中心服务端拉取应用的最新配置。
  1. 这是一个fallback机制，为了防止推送机制失效导致配置不更新
  1. 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 - Not Modified
  1. 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定System Property: apollo.refreshInterval来覆盖，单位为分钟。
3. 客户端从Apollo配置中心服务端获取到应用的最新配置后，会保存在内存中
3. 客户端会把从服务端获取到的配置在本地文件系统缓存一份
  1. 在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置
5. 应用程序可以从Apollo客户端获取最新的配置、订阅配置更新通知

<a name="Jbc5y"></a>
## 二、Http Long Polling
关于实时性，我们看到上面方案中，其实最关键的就是Http Long Polling。<br />那么Http的Long Polling是什么个原理呢？
<a name="JsZIA"></a>
### 2.1.1 定义
长轮询(Long Polling)的服务其客户端是不做轮询的，客户端在发起一次请求后立即挂起，一直到服务器端有更新的时候，服务器才会主动推送信息到客户端。 在服务器端有更新并推送信息过来之前这个周期内，客户端不会有新的多余的请求发生，服务器端对此客户端也啥都不用干，只保留最基本的连接信息，一旦服务器有更新将推送给客户端，客户端将相应的做出处理，处理完后再重新发起下一轮请求。

大概意思：就是客户端发起了一个请求，然后服务器挂起，然后有数据返回了，则返回给客户端并断开。然后客户端重新这个操作。

<a name="TxJ7Z"></a>
### 2.1.2 关注点
上面的概括中，我们需要特别关注的是：

1. 服务端会阻塞请求直到有数据传递或超时才返回；
1. 服务端如何决定自己应不应该返回数据，最终断开这次连接。
1. 客户端响应处理函数会在处理完服务器返回的信息后，再次发出请求，重新建立连接；
1. 当客户端在处理返回数据的时候，服务端可能已经有新的数据到达。

<a name="inWps"></a>
### 2.1.3 分析阶段
我们用nodejs来写个代码分析一下。

<a name="jB0T1"></a>
#### 2.1.3.1 关注点1
关注点1比较好实现。<br />我们利用node来写一个延时返回的代码:

```javascript
const koa = require('koa');

const app = new koa();

app.use(async (ctx, next)=>{
    return new Promise((resolve, reject)=>{
        setTimeout(()=>{
            ctx.body = 'hello world';
            resolve()
        }, 5000);
    })
})

app.listen(8000);
```
我们通过setTimeout延迟请求返回的时间，过了5秒后，我们再返回。<br />这个就是我们关注点一的实现。

<a name="LCiIs"></a>
#### 2.1.3.2 关注点2
关注点2，就是Http long polling最关键的一步：<br />如果我们每次客户端来请求了，就直接返回，那我们的代码就变成了Polling了，就是客户端定时来调用服务端的接口以达到更新配置的情况，也就无法达到实时性的一个情况了，毕竟定时获取这个定时器是有时间的。另外还可能变成死循环，怎么说呢，因为关注点3，客户端响应处理完后，会再次发起。

错误点1：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557387429755-babe3c97-9252-436b-aae4-798a4c0fc912.png#align=left&display=inline&height=253&name=image.png&originHeight=253&originWidth=671&size=14777&status=done&width=671)<br />这个时候，这种其实是Polling模式。<br />缺点：

1. 比如我们定时器是1分钟一次，如果我们在配置中心上更改的配置，要在小于等于一分钟才能收到，那就达不到准实时了。
1. 有人会将定时器设置的比较端，比如5s钟一次，那想象一下，整个事业部这么多系统连接了配置中心，超级多的服务这么调用他，那对于配置中心压力也是够大的。而且配置中心修改，要在小于等于5s的时间才能收到，也没有准实时。

错误点2：<br />没有关注Http long polling机制的条件，就是服务端如何决定自己要不要返回数据。<br />其实是依赖于客户端返回的参数的。<br />否则如果我们给的参数不对，那结果就是，我们的代码会陷入一个死循环。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557387786789-a6e5dbd5-5ac1-4a4d-894f-bc3d048bb189.png#align=left&display=inline&height=286&name=image.png&originHeight=286&originWidth=687&size=12168&status=done&width=687)<br />客户端发起了请求，服务端返回了。然后客户端处理完了，然后再发起请求，服务端又立马返回了，如此反复。<br />会把配置中心搞死的。

所以这块我们用node程序来写一个模拟程序。

```typescript
import koa, {Context} from 'koa';
import Router from 'koa-router';
import Event from 'events';

const router = new Router();

const app = new koa();

let i = 1;
let queue: any[] = [];

router.get("/hello", async (ctx: Context, next)=>{
    ctx.query.num = parseInt(ctx.query.num)
    if(ctx.query.num === -1 || ctx.query.num !== i){
        return ctx.body = i;
    }else{
        let event = new Event();
        queue.push({
            ctx,
            event
        });
        ctx.socket.on('close', ()=>{
            console.log("===>close")
            event.emit('end');
        })
        return new Promise((resolve, reject)=>{
            event.on('send', ()=>{
                ctx.body = i;
                resolve('123');
            });
            event.on('end', ()=>{
                queue.filter((item, index)=>{
                    queue.splice(index, 1);
                })
                resolve({});
            })
        })
    }
})

router.get("/add", async (ctx: Context, next)=>{
    i ++;
    console.log(queue.length)
    queue.map(item=>{
        item.event.emit('send');
    })
    queue = [];
    ctx.body = 'hello world';
})

app.on('abort', ()=>{
    console.log("====>")
})

app.use(router.routes())

app.listen(8000);
```
这块我写了一个typescript的程序来描述一下。

步骤一：<br />当我们发起一个请求的时候，一般最开始我们由于不知道服务器端现在什么配置了，就会先告诉服务器端请给我来一份全部的配置文件。这时候我们的参数是 -1，此处请求是http://127.0.0.1:8000/hello?num=-1<br />然后服务端收到后，就立马返回给我们最新的配置。完全不用等待。<br />返回给我们: 

```
1
```

步骤二：<br />这个时候，我们应该是要拿着刚刚返回的1，去请求新的配置了，这时候我们的请求是<br />http://127.0.0.1:8000/hello?num=1<br />这个时候，服务端根据第14 行的条件都不满足，所以进入下一个else里面，<br />这个时候，服务端不返回，所以用postman看到的情况就是：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557388316806-e566199b-ebc7-44e7-b30a-d129878bf13d.png#align=left&display=inline&height=404&name=image.png&originHeight=404&originWidth=967&size=32059&status=done&width=967)<br />这个时候，服务端就一直不返回了。<br />然后当我们发一个<br />http://127.0.0.1:8000/add的接口后，由于他会去遍历整个queue里面的event进行emit。同时更新了一下i这个变量。所以这个时候，那边收到了请求后，就会去处理，同时返回给客户端，新的i。可以理解为配置的id变了。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557388879586-2310730f-fe45-429f-acd5-d2459ac71e4d.png#align=left&display=inline&height=463&name=image.png&originHeight=463&originWidth=645&size=27287&status=done&width=645)<br />那客户端会对返回数据做一个处理，然后发起一个新的请求，这个时候，就会带上这个返回给我们的id了。<br />新的请求地址是：http://127.0.0.1:8000?num=2，然后服务端如果这阵子没更新，又会进入下面这个状态。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557388979651-ec8205a9-c687-4589-b8c8-d0e639b5ff0e.png#align=left&display=inline&height=435&name=image.png&originHeight=435&originWidth=982&size=35872&status=done&width=982)<br />但是这个时候，我们看到代码中我们有一个处理，就是因为客户端可能关闭掉这个请求了，或者客户端挂了，超时了等。<br />这个时候，这个时候我们需要处理客户端断开的情况，相当于客户端断开了，我们应该尽早的把资源给释放掉，防止其占用我们资源，此处我们的资源就是queue资源了。<br />所以下面的代码，就可以看到我们关闭了这些资源，减少了queue的资源的浪费，其实ctx等资源也是的。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557389198502-d04f8427-53dc-4b3c-a313-1a3e30705cb6.png#align=left&display=inline&height=338&name=image.png&originHeight=338&originWidth=552&size=43298&status=done&width=552)

<a name="v55e0"></a>
#### 2.1.3.3 关注点3
关注点三，就是客户端层面的了，我们需要根据在请求返回后，处理一下，然后重新立马发起，而不要出现内部有什么setTimeout的情况，以及参数需要修改。

<a name="7PK7U"></a>
#### 2.1.3.4 关注点4
这个事情，其实是这样的，就是为了下面这种情况<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557389534987-c37b1528-ecf4-4e9e-8e9d-5d9c1b877753.png#align=left&display=inline&height=308&name=image.png&originHeight=308&originWidth=970&size=26519&status=done&width=970)<br />就是当我们在处理请求的期间，然后服务端又更新了配置，当我们处理完毕后，会立马发起请求，然后应该是立马接收到这个返回值，然后我们再开始处理返回，当处理完毕后，再发起这个请求。

然后这个时候，我们又会进入以下状态：<br />![](https://cdn.nlark.com/yuque/0/2019/png/187105/1557388979651-ec8205a9-c687-4589-b8c8-d0e639b5ff0e.png#align=left&display=inline&height=330&originHeight=435&originWidth=982&status=done&width=746)

<a name="S4kLA"></a>
### 2.1.4 注意点
情况一：<br />由于我们的http long polling会经过nginx，那nginx那层对于长时间没反应的请求，会帮我们做处理，所以我们要注意nginx对于long polling时间的一个设定：

```
try{
	// 此处发起一个long polling
 }catch(e){
   // 情况一：如果我们的请求被nginx给断掉了，那就进入这个里面了
   // 情况二：如果服务端挂了
   // 所以首先我们应该确保我们用Http long polling的时候，nginx那层的时间要大于http long polling的时间
 }
```



<a name="Fq3uV"></a>
## 三、定时获取机制
<a name="vyAG7"></a>
### 3.1 fallback机制
当推送机制出问题的时候，我们可以确保配置不更新，那样也是比较尴尬的事情，所以这个是保险丝，相当于这块会定时拉取。来保障当Http long polling出问题的时候，我们的配置也能正常工作。
<a name="PquWG"></a>
### 3.2 304机制
定时拉取的时候，类似上面我们介绍的，把本地的id告诉服务端，然后服务端会告诉我们，变更了没有，如果没有变更则直接返回http code 304。那我们就不用去拿body了。这样也节约了网络body的流量。因为配置多的话，还是比较浪费，以及，服务端，可以根据id做判断是否变了，如果变了再去数据库获取，那节约对数据库的消耗。毕竟id什么的放在cache或者redis里面还是很快的。
<a name="d2Y0V"></a>
### 3.3 时间频率
上面官方那张图，其实写着java那边相当于5分钟拉取一次，那我们node也可以这么实现。

<a name="RZCoD"></a>
## 四、实时内存机制
<a name="SRWeg"></a>
### 4.1 实时内存
其实我们网上搜到的那个看似对接apollo的sdk：[https://github.com/Quinton/node-apollo](https://github.com/Quinton/node-apollo)，其实Http long polling是没有的，也没法做到实时内存机制。<br />这个相当于，当我们感知到配置更新的时候，我们应该是能同步到node的应用中，当controller或者service拿取的时候，我们应该是拿到新的配置。

<a name="Kq7LU"></a>
### 4.2 其他情形
部分配置修改，可能会导致

1. 重启应用程序
1. url改了，我们可能根据这个url要去另外一个服务那边取配置
1. 等

所以我们暴露了ApolloConfigChangeListener机制，这块的话，相当于可以交给写应用的人去判断，某些配置修改，会让他不得不重启应用，哪些配置更改了，要去别的地方联动其他一些配置，或者告知其他服务等操作。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557390264174-9c1611da-f3ed-457e-b7f5-66bd53eb46d5.png#align=left&display=inline&height=418&name=image.png&originHeight=418&originWidth=444&size=67052&status=done&width=444)

<a name="tVuC6"></a>
## 五、总结
此思路适用于其他语言，比如golang要对接配置中心的话，也可以用上面的机制来实现以下。代码不多，只是有一些注意点，大家要注意。

Http long polling的应用。<br />像WebQQ、Comment都用到长轮询技术，另外一些使用Pull模式消费的消息系统，都会使用Long Polling技术进行优化。

