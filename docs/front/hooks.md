---
title: ahooks理解
---

### useRequest 源码分析

基础功能：

1. 默认请求。
2. 手动触发。
3. 四个生命周期，onBefore,onSuccess,onError,onFinally.
4. 刷新。refresh，refreshAsync。
5. 立即变更数据。
6. 取消请求。

```js
//基础配置api
const {
  loading: boolean,
  data?: TData,
  error?: Error,
  params: TParams || [],
  run: (...params: TParams) => void,
  runAsync: (...params: TParams) => Promise<TData>,
  refresh: () => void,
  refreshAsync: () => Promise<TData>,
  mutate: (data?: TData | ((oldData?: TData) => (TData | undefined))) => void,
  cancel: () => void,
} = useRequest<TData, TParams>(
  service: (...args: TParams) => Promise<TData>,
  {
    manual?: boolean,
    defaultParams?: TParams,
    onBefore?: (params: TParams) => void,
    onSuccess?: (data: TData, params: TParams) => void,
    onError?: (e: Error, params: TParams) => void,
    onFinally?: (params: TParams, data?: TData, e?: Error) => void,
  }
);
```

除开基础功能外，useRequest 支持额外的插件机制，其内部也通过高阶函数的使用，进行了一些插件的整合，如下：

```js
//基础文件，采用一个高阶函数，去加载一些内部已经定义好的plugin，提高了hooks的可扩展性。
function useRequest<TData, TParams extends any[]>(
  service: Service<TData, TParams>,
  options?: Options<TData, TParams>,
  plugins?: Plugin<TData, TParams>[],
) {
  return useRequestImplement<TData, TParams>(service, options, [
    ...(plugins || []),
    useDebouncePlugin,//防抖
    useLoadingDelayPlugin,//延迟loading状态的变化
    usePollingPlugin,//轮询
    useRefreshOnWindowFocusPlugin,//屏幕聚焦时重新请求
    useThrottlePlugin,//节流
    useAutoRunPlugin,//ready字段，依赖刷新
    //缓存，支持设置数据回收时间以及数据新鲜时间。
    //回收时间：该时间过后会从缓存中删除该内容。
    //新鲜时间：第二次会先从缓存中拿数据，再去判断是否过了新鲜时间，若过了再去发起请求，并且进行data和缓存的更新。
    useCachePlugin,
    useRetryPlugin,//错误重试
  ] as Plugin<TData, TParams>[]);
}
```

对于 plugins ：

- hook 参数：fetch 实例，用户配置 option
- 可使用的生命周期：onInit、onBefore、onRequest、onSuccess、onFinally、onError、onFinally、onCancel、onMutate。
  - onInit：规定 fetch 的初始参数。
  - onBefore：通知和修改 state。
  - onRequest：返回包含 servicePromise 的 Promise 对象，替换默认的 service 的调用。
  - onSuccess、onFinally、onError、onFinally、onCancel、onMutate：仅起到通知的效果。

##### useCachePlugin

其中 _useCachePlugin_ 插件略显复杂一些，内部有 cache、cachePromise、cacheSubscribe，这些插件。

- cache 通过一个全局的 map 对象，控制缓存的存取以及设置缓存过期时间和手动清除缓存。
- cachePromise 通过一个全局的 map 对象，根据 cachekey 对 Promise 对象进行缓存，避免不同组件的相同的请求重复发出。
- cacheSubscribe 根据 cachekey 设置数据监听，一旦缓存数据更新，一定会更新到全部依赖于此缓存的对应组件。

### 场景使用

- useAntdTable，配合 antd 的 table 使用的 hook。
- useFunsionTable，实现 FusionForm 和 FusionTable 的联动逻辑。
- useInfiniteScroll，实现常见的无限滚动逻辑。
- usePagination，实现分页逻辑。
- useDynamicList，实现管理动态列表的逻辑，主要是 key。
- useVirtualList，实现虚拟滚动列表的逻辑。
- useHistoryTravel，实现管理历史状态的逻辑。
- useNetwork，实现网络连接状态的逻辑。
- useSelections，实现常见的 Checkbox 的逻辑。
- useCountDown，实现管理倒计时的管理逻辑。
- useCounter，实现管理计数器的逻辑。
- useTextSelection，实现获取用户当前选取的文本内容及位置的逻辑。
- useWebSocket，实现 websocket 的调用逻辑。

### 生命周期

- useMount，实现组件初始化的 hook。
- useUnmount，实现组件卸载时的 hook。
- useUnmountedRef，获取当前组件是否已经卸载的 hook。

### 状态管理

- useSetState， 管理 object 类型的 state，基本与类组件的 setstate 一致。
- useBoolean， 管理 boolean 状态。
- useToggle， 管理在两个状态值之间切换。
- useUrlState， 通过 url query 来管理 state。
- useCookieState， 和 cookie 同步状态的 hook。
- useLocalStorageState 和 localstorage 同步状态的 hook。
- useSessionStorageState，和 sessionStorage 同步状态的 hook。
- useDebounce，防抖状态的 hook。
- useThrottle，节流状态的 hook。
- useMap，管理 map 类型状态。
- useSet，管理 set 类型状态。
- usePrevious，记录上一次状态的 hook。
- useRafState，只在 requestAnimationFrame callback 时更新 state，一般用于性能优化。
- useSafeState， 在组件卸载后异步回调内的 setState 不再执行，避免因组件卸载后更新状态而导致的内存泄漏。
- useGetState，在 usestate 基础上增加 getter，获取最新值。

### 副作用

- useUpdateEffect，等同于与 useEffect，但是第一次不执行。
- useUpdateLayoutEffect，等同于 useLayoutEffect，但是第一次不执行。
-

### useCreation 源码分析

### useReactive 源码分析

作用：提供一种数据响应式的操作体验，定义数据状态不需要写 useState，直接修改属性即可刷新视图。

> [接口网站](https://ahooks.js.org/zh-CN/hooks/use-reactive)

实现步骤：

1. 创建一个 ref。
2. 创建一个可以手动触发组件更新的 hook。
3. 给 ref.current 对象的 get、set、delete 添加代理，其中触发手动更新。

利用了两个全局的 WeekMap,实现代理函数 observer 的性能优化。

```js
// k:v 原对象:代理过的对象
const proxyMap = new WeakMap();
// k:v 代理过的对象:原对象
const rawMap = new WeakMap();
function observer<T extends Record<string, any>>(initialVal: T, cb: () => void): T {
  const existingProxy = proxyMap.get(initialVal);

  // 添加缓存 防止重新构建proxy
  if (existingProxy) {
    return existingProxy;
  }

  // 防止代理已经代理过的对象
  // https://github.com/alibaba/hooks/issues/839
  if (rawMap.has(initialVal)) {
    return initialVal;
  }

  const proxy = new Proxy<T>(initialVal, {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver);
      return isObject(res) ? observer(res, cb) : Reflect.get(target, key);
    },
    set(target, key, val) {
      const ret = Reflect.set(target, key, val);
      cb();
      return ret;
    },
    deleteProperty(target, key) {
      const ret = Reflect.deleteProperty(target, key);
      cb();
      return ret;
    },
  });

  proxyMap.set(initialVal, proxy);
  rawMap.set(proxy, initialVal);

  return proxy;
}
```

> tip: observer 函数会将一个对象封装为对象的代理，当一个复杂对象第一次被 observer 创建对象的代理时，其深层次的子属性仍然是普通对象，但一点复杂对象内部结构被手动修改后，根据 set 函数中的内容其已经被修改为 proxy 对象，所以必须有 rawMap 的存在，否则仅仅一个 existingProxy 的存在会导致复杂对象中子属性对应的对象被反复代理。
