---
title: Pandora源码-Pandora-dashboard(一)
cover: https://img.zcool.cn/community/011e475b86a540a80120245c336724.jpg@1280w_1l_2o_100sh.jpg
authorNick: 金炳
---

## 简介
Pandora是阿里，一个可管理、可度量、可追踪的Node.js应用管理器。文档地址：http://www.midwayjs.org/pandora/zh-cn/ ，
仓库地址：https://github.com/midwayjs/pandora/ 和  https://github.com/midwayjs/pandora-dashboard。

## Pandora-dashboard介绍
这个是对应的Pandora的一个web应用，用来查看pandora应用管理器里面应用的情况。运行方法:
```text
$ npm i pandora-dashboard -g # 全局安装，会全局注册一个命令 pandora-dashboard-dir
$ pandora start --name dashboard `pandora-dashboard-dir` # 使用该命令获得路径，用于启动
```

然后访问网址: http://127.0.0.1:9081/, 他的网页效果：

![](https://user-images.githubusercontent.com/6525544/44789504-4c363100-abcf-11e8-9066-6501928f257a.png)

浏览器通过9801端口访问这个dashboard，Dashboard内部的结构总体如下图：

![](https://user-images.githubusercontent.com/6525544/44789098-54da3780-abce-11e8-8814-e2eec12211ff.png)

然后这个代码比较简单，总体是一个typescript写的一个koa程序，然后项目结构如下图:

![](https://user-images.githubusercontent.com/6525544/44789590-8d2e4580-abcf-11e8-8b0e-6e839bab9254.png)

Impl文件夹内部是对应的router，然后Home会去取上面的html相关的，static取对应的js，css相关的。然后Actuator.ts则是调用7002端口里面的信息。
Stdout，DebuggerProxy都新建了对应的websocket跟后端进行通信输出对应信息。

## 总结
总体我们看到dashboard这层，不像PM2那边的server是做了中心化存储，可以在中心进行查看各机器的信息。而Pandora当前是单机器装对应的Pandora和Pandora-dashboard，无法关于多台机器上面的Pandora的应用。
