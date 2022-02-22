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

> 钩子可以根据执行机制分成 `同步钩子` 和 `异步钩子`。也可以根据功能分成 `基本钩子`、`熔断钩子`、`瀑布钩子`、`循环钩子`。其中异步钩子也分为`异步串行`和`异步并行`。

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
  AsyncParallelHook, //异步并行
  AsyncParallelBailHook, //异步并行
  AsyncSeriesHook, //异步串行
  AsyncSeriesBailHook, //异步串行
  AsyncSeriesLoopHook, //异步串行
  AsyncSeriesWaterfallHook, //异步串行
  HookMap,
  MultiHook,
} = require('tapable');

class Car {
  constructor() {
    this.hooks = {
      accelerate: new SyncHook(['newSpeed']),
      brake: new SyncHook(),
      calculateRoutes: new AsyncParallelHook([
        'source',
        'target',
        'routesList',
      ]),
    };
  }
  setSpeed(newSpeed) {
    // call(xx) 传参调用同步钩子的API
    this.hooks.accelerate.call(newSpeed);
  }

  useNavigationSystemPromise(source, target) {
    const routesList = new List();
    // 调用promise钩子(钩子返回一个promise)的API
    return this.hooks.calculateRoutes
      .promise(source, target, routesList)
      .then(() => {
        return routesList.getRoutes();
      });
  }

  useNavigationSystemAsync(source, target, callback) {
    const routesList = new List();
    // 调用异步钩子API
    this.hooks.calculateRoutes.callAsync(source, target, routesList, (err) => {
      if (err) return callback(err);
      callback(null, routesList.getRoutes());
    });
  }
}

const myCar = new Car();

//注册事件
//同步钩子绑定事件
myCar.hooks.brake.tap('WarningLampPlugin', () => warningLamp.on());
// promise: 绑定promise钩子的API

//异步钩子绑定事件
myCar.hooks.calculateRoutes.tapPromise(
  'GoogleMapsPlugin',
  (source, target, routesList) => {
    // return a promise
    return google.maps.findRoute(source, target).then((route) => {
      routesList.add(route);
    });
  },
);
// tapAsync:绑定异步钩子的API
myCar.hooks.calculateRoutes.tapAsync(
  'BingMapsPlugin',
  (source, target, routesList, callback) => {
    bing.findRoute(source, target, (err, route) => {
      if (err) return callback(err);
      routesList.add(route);
      // call the callback
      callback();
    });
  },
);
// tap: 绑定同步钩子的API
myCar.hooks.calculateRoutes.tap(
  'CachedRoutesPlugin',
  (source, target, routesList) => {
    const cachedRoute = cache.get(source, target);
    if (cachedRoute) routesList.add(cachedRoute);
  },
);

// 注册一个拦截器
myCar.hooks.calculateRoutes.intercept({
  call: (source, target, routesList) => {
    console.log('Starting to calculate routes');
  },
  register: (tapInfo) => {
    // tapInfo = { type: "promise", name: "GoogleMapsPlugin", fn: ... }
    console.log(`${tapInfo.name} is doing its job`);
    return tapInfo; // may return a new tapInfo object
  },
});
```

# webpack 编译分析

### webpack 启动

```js
let webpack = require('webpack');

let options = require('./webpack.config');

// 在工作的时候需要接收的参数就是webpack的配置, 这个方法返回的结果也就是complier就可以进行编译了
let complier = webpack(options);

// 编译
complier.run((err, stats) => {
  // 错误优先
  console.info(err);
  console.info(stats.toJson());
});
```

> webpack 启动后还会进行添加缓存、配置项初始化等一些操作，不过多叙述，主要看 compile 编译主流程。

### compiler 与 compilation

> compiler 对象是 webpack 的编译器对象，会在 webpack 第一次初始化的时候生成，接受了用户的自定义配置合成对应的参数，我们可以通过 compiler 对象拿到 webpack 的主环境信息。

> compilation 对象代表了一次单一的版本构建和生成资源。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，一次新的编译将被创建，从而生成一组新的编译资源。一个编译对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。编译对象也提供了很多关键点回调供插件做自定义处理时选择使用。

1. entry-option， 初始化配置。
2. beforeRun，编译之前
3. run，开始编译。
4. beforeCompile，
5. compile
6. compilation，创建完成 compilation，并传递给主流程
7. make，从 entry 开始递归分析以来，对每个模块进行 build
8. 对模块进行解析。
9. build-module， 开始构建某个模块。
10. 加载 loader，对模块进行编译，生成 ast 树。（暂时没看到在哪一步）
11. program，收集依赖。
12. seal， build 完成后进行优化。
13. emit， 对构建完成内容进行输出到 dist。

Compiler 中的钩子

```js
this.hooks = {
  /** @type {SyncHook<[]>} */
  initialize: new SyncHook([]),

  /** @type {SyncBailHook<[Compilation], boolean>} */
  shouldEmit: new SyncBailHook(['compilation']),
  /** @type {AsyncSeriesHook<[Stats]>} */
  done: new AsyncSeriesHook(['stats']),
  /** @type {SyncHook<[Stats]>} */
  afterDone: new SyncHook(['stats']),
  /** @type {AsyncSeriesHook<[]>} */
  additionalPass: new AsyncSeriesHook([]),
  /** @type {AsyncSeriesHook<[Compiler]>} */
  beforeRun: new AsyncSeriesHook(['compiler']),
  /** @type {AsyncSeriesHook<[Compiler]>} */
  run: new AsyncSeriesHook(['compiler']),
  /** @type {AsyncSeriesHook<[Compilation]>} */
  emit: new AsyncSeriesHook(['compilation']),
  /** @type {AsyncSeriesHook<[string, AssetEmittedInfo]>} */
  assetEmitted: new AsyncSeriesHook(['file', 'info']),
  /** @type {AsyncSeriesHook<[Compilation]>} */
  afterEmit: new AsyncSeriesHook(['compilation']),

  /** @type {SyncHook<[Compilation, CompilationParams]>} */
  thisCompilation: new SyncHook(['compilation', 'params']),
  /** @type {SyncHook<[Compilation, CompilationParams]>} */
  compilation: new SyncHook(['compilation', 'params']),
  /** @type {SyncHook<[NormalModuleFactory]>} */
  normalModuleFactory: new SyncHook(['normalModuleFactory']),
  /** @type {SyncHook<[ContextModuleFactory]>}  */
  contextModuleFactory: new SyncHook(['contextModuleFactory']),

  /** @type {AsyncSeriesHook<[CompilationParams]>} */
  beforeCompile: new AsyncSeriesHook(['params']),
  /** @type {SyncHook<[CompilationParams]>} */
  compile: new SyncHook(['params']),
  /** @type {AsyncParallelHook<[Compilation]>} */
  make: new AsyncParallelHook(['compilation']),
  /** @type {AsyncParallelHook<[Compilation]>} */
  finishMake: new AsyncSeriesHook(['compilation']),
  /** @type {AsyncSeriesHook<[Compilation]>} */
  afterCompile: new AsyncSeriesHook(['compilation']),

  /** @type {AsyncSeriesHook<[Compiler]>} */
  watchRun: new AsyncSeriesHook(['compiler']),
  /** @type {SyncHook<[Error]>} */
  failed: new SyncHook(['error']),
  /** @type {SyncHook<[string | null, number]>} */
  invalid: new SyncHook(['filename', 'changeTime']),
  /** @type {SyncHook<[]>} */
  watchClose: new SyncHook([]),
  /** @type {AsyncSeriesHook<[]>} */
  shutdown: new AsyncSeriesHook([]),

  /** @type {SyncBailHook<[string, string, any[]], true>} */
  infrastructureLog: new SyncBailHook(['origin', 'type', 'args']),

  // TODO the following hooks are weirdly located here
  // TODO move them for webpack 5
  /** @type {SyncHook<[]>} */
  environment: new SyncHook([]),
  /** @type {SyncHook<[]>} */
  afterEnvironment: new SyncHook([]),
  /** @type {SyncHook<[Compiler]>} */
  afterPlugins: new SyncHook(['compiler']),
  /** @type {SyncHook<[Compiler]>} */
  afterResolvers: new SyncHook(['compiler']),
  /** @type {SyncBailHook<[string, Entry], boolean>} */
  entryOption: new SyncBailHook(['context', 'entry']),
};
```

Compilation 中的钩子

```js
this.hooks = {
  /** @type {SyncHook<[Module]>} */
  buildModule: new SyncHook(['module']),
  /** @type {SyncHook<[Module]>} */
  rebuildModule: new SyncHook(['module']),
  /** @type {SyncHook<[Module, WebpackError]>} */
  failedModule: new SyncHook(['module', 'error']),
  /** @type {SyncHook<[Module]>} */
  succeedModule: new SyncHook(['module']),
  /** @type {SyncHook<[Module]>} */
  stillValidModule: new SyncHook(['module']),

  /** @type {SyncHook<[Dependency, EntryOptions]>} */
  addEntry: new SyncHook(['entry', 'options']),
  /** @type {SyncHook<[Dependency, EntryOptions, Error]>} */
  failedEntry: new SyncHook(['entry', 'options', 'error']),
  /** @type {SyncHook<[Dependency, EntryOptions, Module]>} */
  succeedEntry: new SyncHook(['entry', 'options', 'module']),

  /** @type {SyncWaterfallHook<[(string[] | ReferencedExport)[], Dependency, RuntimeSpec]>} */
  dependencyReferencedExports: new SyncWaterfallHook([
    'referencedExports',
    'dependency',
    'runtime',
  ]),

  /** @type {SyncHook<[ExecuteModuleArgument, ExecuteModuleContext]>} */
  executeModule: new SyncHook(['options', 'context']),
  /** @type {AsyncParallelHook<[ExecuteModuleArgument, ExecuteModuleContext]>} */
  prepareModuleExecution: new AsyncParallelHook(['options', 'context']),

  /** @type {AsyncSeriesHook<[Iterable<Module>]>} */
  finishModules: new AsyncSeriesHook(['modules']),
  /** @type {AsyncSeriesHook<[Module]>} */
  finishRebuildingModule: new AsyncSeriesHook(['module']),
  /** @type {SyncHook<[]>} */
  unseal: new SyncHook([]),
  /** @type {SyncHook<[]>} */
  seal: new SyncHook([]),

  /** @type {SyncHook<[]>} */
  beforeChunks: new SyncHook([]),
  /** @type {SyncHook<[Iterable<Chunk>]>} */
  afterChunks: new SyncHook(['chunks']),

  /** @type {SyncBailHook<[Iterable<Module>]>} */
  optimizeDependencies: new SyncBailHook(['modules']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  afterOptimizeDependencies: new SyncHook(['modules']),

  /** @type {SyncHook<[]>} */
  optimize: new SyncHook([]),
  /** @type {SyncBailHook<[Iterable<Module>]>} */
  optimizeModules: new SyncBailHook(['modules']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  afterOptimizeModules: new SyncHook(['modules']),

  /** @type {SyncBailHook<[Iterable<Chunk>, ChunkGroup[]]>} */
  optimizeChunks: new SyncBailHook(['chunks', 'chunkGroups']),
  /** @type {SyncHook<[Iterable<Chunk>, ChunkGroup[]]>} */
  afterOptimizeChunks: new SyncHook(['chunks', 'chunkGroups']),

  /** @type {AsyncSeriesHook<[Iterable<Chunk>, Iterable<Module>]>} */
  optimizeTree: new AsyncSeriesHook(['chunks', 'modules']),
  /** @type {SyncHook<[Iterable<Chunk>, Iterable<Module>]>} */
  afterOptimizeTree: new SyncHook(['chunks', 'modules']),

  /** @type {AsyncSeriesBailHook<[Iterable<Chunk>, Iterable<Module>]>} */
  optimizeChunkModules: new AsyncSeriesBailHook(['chunks', 'modules']),
  /** @type {SyncHook<[Iterable<Chunk>, Iterable<Module>]>} */
  afterOptimizeChunkModules: new SyncHook(['chunks', 'modules']),
  /** @type {SyncBailHook<[], boolean>} */
  shouldRecord: new SyncBailHook([]),

  /** @type {SyncHook<[Chunk, Set<string>, RuntimeRequirementsContext]>} */
  additionalChunkRuntimeRequirements: new SyncHook([
    'chunk',
    'runtimeRequirements',
    'context',
  ]),
  /** @type {HookMap<SyncBailHook<[Chunk, Set<string>, RuntimeRequirementsContext]>>} */
  runtimeRequirementInChunk: new HookMap(
    () => new SyncBailHook(['chunk', 'runtimeRequirements', 'context']),
  ),
  /** @type {SyncHook<[Module, Set<string>, RuntimeRequirementsContext]>} */
  additionalModuleRuntimeRequirements: new SyncHook([
    'module',
    'runtimeRequirements',
    'context',
  ]),
  /** @type {HookMap<SyncBailHook<[Module, Set<string>, RuntimeRequirementsContext]>>} */
  runtimeRequirementInModule: new HookMap(
    () => new SyncBailHook(['module', 'runtimeRequirements', 'context']),
  ),
  /** @type {SyncHook<[Chunk, Set<string>, RuntimeRequirementsContext]>} */
  additionalTreeRuntimeRequirements: new SyncHook([
    'chunk',
    'runtimeRequirements',
    'context',
  ]),
  /** @type {HookMap<SyncBailHook<[Chunk, Set<string>, RuntimeRequirementsContext]>>} */
  runtimeRequirementInTree: new HookMap(
    () => new SyncBailHook(['chunk', 'runtimeRequirements', 'context']),
  ),

  /** @type {SyncHook<[RuntimeModule, Chunk]>} */
  runtimeModule: new SyncHook(['module', 'chunk']),

  /** @type {SyncHook<[Iterable<Module>, any]>} */
  reviveModules: new SyncHook(['modules', 'records']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  beforeModuleIds: new SyncHook(['modules']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  moduleIds: new SyncHook(['modules']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  optimizeModuleIds: new SyncHook(['modules']),
  /** @type {SyncHook<[Iterable<Module>]>} */
  afterOptimizeModuleIds: new SyncHook(['modules']),

  /** @type {SyncHook<[Iterable<Chunk>, any]>} */
  reviveChunks: new SyncHook(['chunks', 'records']),
  /** @type {SyncHook<[Iterable<Chunk>]>} */
  beforeChunkIds: new SyncHook(['chunks']),
  /** @type {SyncHook<[Iterable<Chunk>]>} */
  chunkIds: new SyncHook(['chunks']),
  /** @type {SyncHook<[Iterable<Chunk>]>} */
  optimizeChunkIds: new SyncHook(['chunks']),
  /** @type {SyncHook<[Iterable<Chunk>]>} */
  afterOptimizeChunkIds: new SyncHook(['chunks']),

  /** @type {SyncHook<[Iterable<Module>, any]>} */
  recordModules: new SyncHook(['modules', 'records']),
  /** @type {SyncHook<[Iterable<Chunk>, any]>} */
  recordChunks: new SyncHook(['chunks', 'records']),

  /** @type {SyncHook<[Iterable<Module>]>} */
  optimizeCodeGeneration: new SyncHook(['modules']),

  /** @type {SyncHook<[]>} */
  beforeModuleHash: new SyncHook([]),
  /** @type {SyncHook<[]>} */
  afterModuleHash: new SyncHook([]),

  /** @type {SyncHook<[]>} */
  beforeCodeGeneration: new SyncHook([]),
  /** @type {SyncHook<[]>} */
  afterCodeGeneration: new SyncHook([]),

  /** @type {SyncHook<[]>} */
  beforeRuntimeRequirements: new SyncHook([]),
  /** @type {SyncHook<[]>} */
  afterRuntimeRequirements: new SyncHook([]),

  /** @type {SyncHook<[]>} */
  beforeHash: new SyncHook([]),
  /** @type {SyncHook<[Chunk]>} */
  contentHash: new SyncHook(['chunk']),
  /** @type {SyncHook<[]>} */
  afterHash: new SyncHook([]),
  /** @type {SyncHook<[any]>} */
  recordHash: new SyncHook(['records']),
  /** @type {SyncHook<[Compilation, any]>} */
  record: new SyncHook(['compilation', 'records']),

  /** @type {SyncHook<[]>} */
  beforeModuleAssets: new SyncHook([]),
  /** @type {SyncBailHook<[], boolean>} */
  shouldGenerateChunkAssets: new SyncBailHook([]),
  /** @type {SyncHook<[]>} */
  beforeChunkAssets: new SyncHook([]),
  // TODO webpack 6 remove
  /** @deprecated */
  additionalChunkAssets: createProcessAssetsHook(
    'additionalChunkAssets',
    Compilation.PROCESS_ASSETS_STAGE_ADDITIONAL,
    () => [this.chunks],
    'DEP_WEBPACK_COMPILATION_ADDITIONAL_CHUNK_ASSETS',
  ),

  // TODO webpack 6 deprecate
  /** @deprecated */
  additionalAssets: createProcessAssetsHook(
    'additionalAssets',
    Compilation.PROCESS_ASSETS_STAGE_ADDITIONAL,
    () => [],
  ),
  // TODO webpack 6 remove
  /** @deprecated */
  optimizeChunkAssets: createProcessAssetsHook(
    'optimizeChunkAssets',
    Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE,
    () => [this.chunks],
    'DEP_WEBPACK_COMPILATION_OPTIMIZE_CHUNK_ASSETS',
  ),
  // TODO webpack 6 remove
  /** @deprecated */
  afterOptimizeChunkAssets: createProcessAssetsHook(
    'afterOptimizeChunkAssets',
    Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE + 1,
    () => [this.chunks],
    'DEP_WEBPACK_COMPILATION_AFTER_OPTIMIZE_CHUNK_ASSETS',
  ),
  // TODO webpack 6 deprecate
  /** @deprecated */
  optimizeAssets: processAssetsHook,
  // TODO webpack 6 deprecate
  /** @deprecated */
  afterOptimizeAssets: afterProcessAssetsHook,

  processAssets: processAssetsHook,
  afterProcessAssets: afterProcessAssetsHook,
  /** @type {AsyncSeriesHook<[CompilationAssets]>} */
  processAdditionalAssets: new AsyncSeriesHook(['assets']),

  /** @type {SyncBailHook<[], boolean>} */
  needAdditionalSeal: new SyncBailHook([]),
  /** @type {AsyncSeriesHook<[]>} */
  afterSeal: new AsyncSeriesHook([]),

  /** @type {SyncWaterfallHook<[RenderManifestEntry[], RenderManifestOptions]>} */
  renderManifest: new SyncWaterfallHook(['result', 'options']),

  /** @type {SyncHook<[Hash]>} */
  fullHash: new SyncHook(['hash']),
  /** @type {SyncHook<[Chunk, Hash, ChunkHashContext]>} */
  chunkHash: new SyncHook(['chunk', 'chunkHash', 'ChunkHashContext']),

  /** @type {SyncHook<[Module, string]>} */
  moduleAsset: new SyncHook(['module', 'filename']),
  /** @type {SyncHook<[Chunk, string]>} */
  chunkAsset: new SyncHook(['chunk', 'filename']),

  /** @type {SyncWaterfallHook<[string, object, AssetInfo]>} */
  assetPath: new SyncWaterfallHook(['path', 'options', 'assetInfo']),

  /** @type {SyncBailHook<[], boolean>} */
  needAdditionalPass: new SyncBailHook([]),

  /** @type {SyncHook<[Compiler, string, number]>} */
  childCompiler: new SyncHook([
    'childCompiler',
    'compilerName',
    'compilerIndex',
  ]),

  /** @type {SyncBailHook<[string, LogEntry], true>} */
  log: new SyncBailHook(['origin', 'logEntry']),

  /** @type {SyncWaterfallHook<[WebpackError[]]>} */
  processWarnings: new SyncWaterfallHook(['warnings']),
  /** @type {SyncWaterfallHook<[WebpackError[]]>} */
  processErrors: new SyncWaterfallHook(['errors']),

  /** @type {HookMap<SyncHook<[Partial<NormalizedStatsOptions>, CreateStatsOptionsContext]>>} */
  statsPreset: new HookMap(() => new SyncHook(['options', 'context'])),
  /** @type {SyncHook<[Partial<NormalizedStatsOptions>, CreateStatsOptionsContext]>} */
  statsNormalize: new SyncHook(['options', 'context']),
  /** @type {SyncHook<[StatsFactory, NormalizedStatsOptions]>} */
  statsFactory: new SyncHook(['statsFactory', 'options']),
  /** @type {SyncHook<[StatsPrinter, NormalizedStatsOptions]>} */
  statsPrinter: new SyncHook(['statsPrinter', 'options']),

  get normalModuleLoader() {
    return getNormalModuleLoader();
  },
};
```

> 其中很多具体的细节实现，还没有了解完全，当决定要实现一个 plugins，主要从各种钩子中拿到自己所需要的 compiler 和 compilation 中的信息，在进行对其重新赋值。类似 webpack 的 loader 就是通过 loaderPlugins 实现功能流程。
