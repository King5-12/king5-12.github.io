---
title: webpack
nav:
  title: 前端
---

# webpack 编译流程

可以分成三个阶段

- 配置初始化
- 内容编译
- 输出编译后的内容

webpack 的整体工作过程可以看成是一个：`事件驱动型事件流工作机制`

# tapable 分析

> tapable 是一个事件流控制器，主要是通过控制钩子函数的发布和订阅进行实现。

## tapable 钩子类型

> 钩子可以根据执行机制分成 `同步钩子` 和 `异步钩子`。也可以根据功能分成 `基本钩子`、`熔断钩子`、`瀑布钩子`、`循环钩子`。

根据功能分类说明：

- Hook:基本钩子，这个钩子只会简单的调用每个注册进去的函数。
- BailHook:熔断钩子，熔断钩子允许提前退出，当任何一个注册进入的函数，其返回值不为`undefined`时，退出后续函数的执行。
- WaterfallHook：瀑布钩子，任何一个注册进去的函数，返回值都会传递到下一个函数。
- LoopHook：循环钩子，在当前执行的已注册的函数中，返回不为 `undefined` 时会一直循环执行当前函数。

```js
const {
  SyncHook,
  SyncBailHook,
  SyncWaterfallHook,
  SyncLoopHook,
  AsyncParallelHook,
  AsyncParallelBailHook,
  AsyncSeriesHook,
  AsyncSeriesBailHook,
  AsyncSeriesLoopHook,
  AsyncSeriesWaterfallHook,
  HookMap,
  MultiHook,
} = require('tapable');
```
