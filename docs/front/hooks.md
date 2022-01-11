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

```ts
//基础文件，采用一个高阶函数，去加载一些内部已经定义好的plugin，提高了hooks的可扩展性。
function useRequest<TData, TParams extends any[]>(
  service: Service<TData, TParams>,
  options?: Options<TData, TParams>,
  plugins?: Plugin<TData, TParams>[],
) {
  return useRequestImplement<TData, TParams>(service, options, [
    ...(plugins || []),
    useDebouncePlugin, //防抖
    useLoadingDelayPlugin, //延迟loading状态的变化
    usePollingPlugin, //轮询
    useRefreshOnWindowFocusPlugin, //屏幕聚焦时重新请求
    useThrottlePlugin, //节流
    useAutoRunPlugin, //ready字段，依赖刷新
    //缓存，支持设置数据回收时间以及数据新鲜时间 SWR(stale-while-revalidate)。
    //回收时间：该时间过后会从缓存中删除该内容。
    //新鲜时间：第二次会先从缓存中拿数据，再去判断是否过了新鲜时间，若过了再去发起请求，并且进行data和缓存的更新。
    useCachePlugin,
    useRetryPlugin, //错误重试
  ] as Plugin<TData, TParams>[]);
}
```

##### useRequestImplement

包含一个 fetch 类实例的创建，以及插件 hook 的加载。

对于 plugins ：

- hook 参数：fetch 实例，用户配置 option
- 可使用的生命周期：onInit、onBefore、onRequest、onSuccess、onFinally、onError、onFinally、onCancel、onMutate。
  - onInit：规定 fetch 的初始参数。
  - onBefore：通知和修改 state。
  - onRequest：返回包含 servicePromise 的 Promise 对象，替换默认的 service 的调用。
  - onSuccess、onFinally、onError、onFinally、onCancel、onMutate：仅起到通知的效果。

useRequestImplement.ts

```ts
function useRequestImplement<TData, TParams extends any[]>(
  service: Service<TData, TParams>,
  options: Options<TData, TParams> = {},
  plugins: Plugin<TData, TParams>[] = [],
) {
  const { manual = false, ...rest } = options;

  const fetchOptions = {
    manual,
    ...rest,
  };
  //内部ref实现获取最新传入的service
  const serviceRef = useLatest(service);

  //主动更新函数
  const update = useUpdate();

  //创建fetch实例
  const fetchInstance = useCreation(() => {
    //创建实例前先加载plugin的onInit生命周期
    const initState = plugins
      .map((p) => p?.onInit?.(fetchOptions))
      .filter(Boolean);

    return new Fetch<TData, TParams>(
      serviceRef,
      fetchOptions,
      update,
      Object.assign({}, ...initState),
    );
  }, []);
  //保证配置更新后类中同步更新
  fetchInstance.options = fetchOptions;
  //执行插件hooks并将其全部挂载到fetch实例中方便fetch内部调用到各个生命周期。
  // run all plugins hooks
  fetchInstance.pluginImpls = plugins.map((p) =>
    p(fetchInstance, fetchOptions),
  );

  //是否自动请求
  useMount(() => {
    if (!manual) {
      // useCachePlugin can set fetchInstance.state.params from cache when init
      const params = fetchInstance.state.params || options.defaultParams || [];
      // @ts-ignore
      fetchInstance.run(...params);
    }
  });

  useUnmount(() => {
    fetchInstance.cancel();
  });

  return {
    loading: fetchInstance.state.loading,
    data: fetchInstance.state.data,
    error: fetchInstance.state.error,
    params: fetchInstance.state.params || [],
    cancel: useMemoizedFn(fetchInstance.cancel.bind(fetchInstance)),
    refresh: useMemoizedFn(fetchInstance.refresh.bind(fetchInstance)),
    refreshAsync: useMemoizedFn(fetchInstance.refreshAsync.bind(fetchInstance)),
    run: useMemoizedFn(fetchInstance.run.bind(fetchInstance)),
    runAsync: useMemoizedFn(fetchInstance.runAsync.bind(fetchInstance)),
    mutate: useMemoizedFn(fetchInstance.mutate.bind(fetchInstance)),
  } as Result<TData, TParams>;
}

export default useRequestImplement;
```

Fetch.ts

```ts
export default class Fetch<TData, TParams extends any[]> {
  pluginImpls: PluginReturn<TData, TParams>[];

  count: number = 0;

  state: FetchState<TData, TParams> = {
    loading: false,
    params: undefined,
    data: undefined,
    error: undefined,
  };

  constructor(
    public serviceRef: MutableRefObject<Service<TData, TParams>>,
    public options: Options<TData, TParams>,
    public subscribe: Subscribe,
    public initState: Partial<FetchState<TData, TParams>> = {},
  ) {
    this.state = {
      ...this.state,
      loading: !options.manual,
      ...initState,
    };
  }

  setState(s: Partial<FetchState<TData, TParams>> = {}) {
    this.state = {
      ...this.state,
      ...s,
    };
    this.subscribe();
  }

  runPluginHandler(event: keyof PluginReturn<TData, TParams>, ...rest: any[]) {
    // 调用插件中返回的生命周期
    const r = this.pluginImpls.map((i) => i[event]?.(...rest)).filter(Boolean);
    return Object.assign({}, ...r);
  }
  //执行请求
  async runAsync(...params: TParams): Promise<TData> {
    //通过count控制请求的取消
    this.count += 1;
    const currentCount = this.count;

    const {
      stopNow = false,
      returnNow = false,
      ...state
    } = this.runPluginHandler('onBefore', params);

    // stop request
    if (stopNow) {
      return new Promise(() => {});
    }

    this.setState({
      loading: true,
      params,
      ...state,
    });

    // return now
    if (returnNow) {
      return Promise.resolve(state.data);
    }
    //用户配置的回调
    this.options.onBefore?.(params);

    try {
      // replace service
      //插件中可以在该生命周期中改写请求的返回的promise
      let { servicePromise } = this.runPluginHandler(
        'onRequest',
        this.serviceRef.current,
        params,
      );

      if (!servicePromise) {
        servicePromise = this.serviceRef.current(...params);
      }

      const res = await servicePromise;

      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      // const formattedResult = this.options.formatResultRef.current ? this.options.formatResultRef.current(res) : res;

      this.setState({
        data: res,
        error: undefined,
        loading: false,
      });

      this.options.onSuccess?.(res, params);
      this.runPluginHandler('onSuccess', res, params);

      this.options.onFinally?.(params, res, undefined);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, res, undefined);
      }

      return res;
    } catch (error) {
      if (currentCount !== this.count) {
        // prevent run.then when request is canceled
        return new Promise(() => {});
      }

      this.setState({
        error,
        loading: false,
      });

      this.options.onError?.(error, params);
      this.runPluginHandler('onError', error, params);

      this.options.onFinally?.(params, undefined, error);

      if (currentCount === this.count) {
        this.runPluginHandler('onFinally', params, undefined, error);
      }

      throw error;
    }
  }

  run(...params: TParams) {
    this.runAsync(...params).catch((error) => {
      if (!this.options.onError) {
        console.error(error);
      }
    });
  }

  cancel() {
    this.count += 1;
    this.setState({
      loading: false,
    });

    this.runPluginHandler('onCancel');
  }

  refresh() {
    // @ts-ignore
    this.run(...(this.state.params || []));
  }

  refreshAsync() {
    // @ts-ignore
    return this.runAsync(...(this.state.params || []));
  }

  //直接修改data状态
  mutate(data?: TData | ((oldData?: TData) => TData | undefined)) {
    let targetData: TData | undefined;
    if (typeof data === 'function') {
      // @ts-ignore
      targetData = data(this.state.data);
    } else {
      targetData = data;
    }

    this.runPluginHandler('onMutate', targetData);

    this.setState({
      data: targetData,
    });
  }
}
```

##### useCachePlugin

其中 _useCachePlugin_ 插件略显复杂一些，内部有 cache、cachePromise、cacheSubscribe，这些插件。

- cache 通过一个全局的 map 对象，控制缓存的存取以及设置缓存过期时间和手动清除缓存。
- cachePromise 通过一个全局的 map 对象，根据 cachekey 对 Promise 对象进行缓存，避免不同组件的相同的请求重复发出。
- cacheSubscribe 根据 cachekey 设置数据监听，一旦缓存数据更新，一定会更新到全部依赖于此缓存的对应组件。

useCachePlugin.ts

```ts
const useCachePlugin: Plugin<any, any[]> = (
  fetchInstance,
  {
    cacheKey,
    cacheTime = 5 * 60 * 1000,
    staleTime = 0,
    setCache: customSetCache,
    getCache: customGetCache,
  },
) => {
  const unSubscribeRef = useRef<() => void>();

  const currentPromiseRef = useRef<Promise<any>>();

  const _setCache = (key: string, cachedData: CachedData) => {
    if (customSetCache) {
      customSetCache(cachedData);
    } else {
      cache.setCache(key, cacheTime, cachedData);
    }
    cacheSubscribe.trigger(key, cachedData.data);
  };

  const _getCache = (key: string, params: any[] = []) => {
    if (customGetCache) {
      return customGetCache(params);
    }
    return cache.getCache(key);
  };

  useCreation(() => {
    if (!cacheKey) {
      return;
    }

    // get data from cache when init
    const cacheData = _getCache(cacheKey);
    //初始化时假如有缓存，则直接更改实例上的状态
    if (cacheData && Object.hasOwnProperty.call(cacheData, 'data')) {
      fetchInstance.state.data = cacheData.data;
      fetchInstance.state.params = cacheData.params;
      if (
        staleTime === -1 ||
        new Date().getTime() - cacheData.time <= staleTime
      ) {
        fetchInstance.state.loading = false;
      }
    }

    // subscribe same cachekey update, trigger update
    //缓存更新时同步修改fetch实例中的状态。
    unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (data) => {
      fetchInstance.setState({ data });
    });
  }, []);

  useUnmount(() => {
    unSubscribeRef.current?.();
  });

  if (!cacheKey) {
    return {};
  }

  return {
    onBefore: (params) => {
      const cacheData = _getCache(cacheKey, params);

      if (!cacheData || !Object.hasOwnProperty.call(cacheData, 'data')) {
        return {};
      }

      // If the data is fresh, stop request
      if (
        staleTime === -1 ||
        new Date().getTime() - cacheData.time <= staleTime
      ) {
        return {
          loading: false,
          data: cacheData?.data,
          returnNow: true,
        };
      } else {
        // If the data is stale, return data, and request continue
        return {
          data: cacheData?.data,
        };
      }
    },
    onRequest: (service, args) => {
      let servicePromise = cachePromise.getCachePromise(cacheKey);

      // If has servicePromise, and is not trigger by self, then use it
      if (servicePromise && servicePromise !== currentPromiseRef.current) {
        return { servicePromise };
      }

      servicePromise = service(...args);
      currentPromiseRef.current = servicePromise;
      cachePromise.setCachePromise(cacheKey, servicePromise);
      return { servicePromise };
    },
    onSuccess: (data, params) => {
      if (cacheKey) {
        // cancel subscribe, avoid trgger self
        unSubscribeRef.current?.();
        _setCache(cacheKey, {
          data,
          params,
          time: new Date().getTime(),
        });
        // resubscribe
        unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (d) => {
          fetchInstance.setState({ data: d });
        });
      }
    },
    onMutate: (data) => {
      if (cacheKey) {
        // cancel subscribe, avoid trgger self
        unSubscribeRef.current?.();
        _setCache(cacheKey, {
          data,
          params: fetchInstance.state.params,
          time: new Date().getTime(),
        });
        // resubscribe
        unSubscribeRef.current = cacheSubscribe.subscribe(cacheKey, (d) => {
          fetchInstance.setState({ data: d });
        });
      }
    },
  };
};
```

### Scene

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

### LifeCycle

- useMount，实现组件初始化的 hook。
- useUnmount，实现组件卸载时的 hook。
- useUnmountedRef，获取当前组件是否已经卸载的 hook。

### State

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

### Effect

- useUpdateEffect，等同于与 useEffect，但是第一次不执行。
- useUpdateLayoutEffect，等同于 useLayoutEffect，但是第一次不执行。
- useAsyncEffect, useEffect 支持异步函数。考虑到了迭代器。
- useDebounceEffect，为 useEffect 增加防抖。
- useDebounceFn，用来处理防抖函数的 hook。
- useThrottleFn，用来处理函数节流的 hook。
- useThrottleEffect，为 useEffect 增加节流。
- useDeepCompareEffect，与 useEffect 一致，deps 进行深比较。
- useInterval，处理 setInterval 的 hook。
- useTimeout，处理 setTimeout 的 hook。
- useLockFn，给异步函数增加静态锁，防止并发执行。
- useUpdate，强制组件更新的 hook。

### Dom

- useEventListener， 使用 addEventListener。
- useClickAway， 监听目标元素外的点击事件。
- useDocumentVisibility，监听页面是否可见。
- useDrop&useDrag,处理元素拖拽的 hook，需要互相搭配。
- useEventTarget，onChange 和 value 逻辑封装。
- useExternal，动态注入 js 或 css 资源。
- useTitle, 用于页面 title 设置。
- useFavicon，用于页面 favicon。
- useFullscreen，管理全屏的 hook。
- useHover，监听 dom 元素是否有鼠标悬停。
- useInViewport，监听元素是否在可见区域，以及元素可见比例。
- useKeyPress，监听键盘按键，支持组合键。
- useLongPress，监听目标元素的长按事件。
- useMouse，监听鼠标位置。
- useResponsive，获取响应式信息。
- useScroll，监听元素的滚动位置。
- useSize，监听 dom 节点尺寸变化。
- useFocusWithin，监听当前焦点是否在某个区域内。

### Advanced

- useControllableValue，管理可受控同时可非受控的组件的状态，父组件传递了受控参数，则受父组件控制，若未传递则自己控制状态。
- useCreation，useMemo 或 useRef 的替代品。
- useEventEmitter，创建一个事件通知实例。
- useIsomorphicLayoutEffect， 在非浏览器环境返回 useEffect，在浏览器环境返回 useLayoutEffect。
- useLatest， 返回当前最新值，避免闭包。
- useMemoizedFn， 持久化 function，代替 useCallback，函数地址永远不会变化。
- useReactive，数据响应式体验，数据直接修改属性即可刷新视图。

### dev

- useTrackedEffect， 追踪是哪个依赖出发了 useEffect 的执行。
- useWhyDidYouUpdate， 帮助开发者排查是那个属性改变导致了组件的 rerender。

### useCreation 源码分析

### useEventEmitter 源码分析

```js
//需求：
//事件通知机制的类。
//需要进行事件订阅以及发送事件通知。
//事件在组件中进行订阅的同时需要在组件卸载时进行事件的取消订阅。

//类中存在react的useRef和useEffect的hook，通过将这个类的实例挂在到一个组件的ref中，又将事件注册的hook，通过传递挂在到组件的实例中，只需要在用的地方将事件进行提交即可。
export class EventEmitter<T> {
  private subscriptions = new Set<Subscription<T>>();

  emit = (val: T) => {
    for (const subscription of this.subscriptions) {
      subscription(val);
    }
  };

  useSubscription = (callback: Subscription<T>) => {
    // eslint-disable-next-line react-hooks/rules-of-hooks
    const callbackRef = useRef<Subscription<T>>();
    callbackRef.current = callback;
    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      function subscription(val: T) {
        if (callbackRef.current) {
          callbackRef.current(val);
        }
      }
      this.subscriptions.add(subscription);
      return () => {
        this.subscriptions.delete(subscription);
      };
    }, []);
  };
}

export default function useEventEmitter<T = void>() {
  const ref = useRef<EventEmitter<T>>();
  if (!ref.current) {
    ref.current = new EventEmitter();
  }
  return ref.current;
}
```

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

### createEffectWithTarget.ts

```ts
const createEffectWithTarget = (
  useEffectType: typeof useEffect | typeof useLayoutEffect,
) => {
  /**
   *
   * @param effect
   * @param deps
   * @param target target should compare ref.current vs ref.current, dom vs dom, ()=>dom vs ()=>dom
   */
  const useEffectWithTarget = (
    effect: EffectCallback,
    deps: DependencyList,
    target: BasicTarget<any> | BasicTarget<any>[],
  ) => {
    const hasInitRef = useRef(false);

    const lastElementRef = useRef<(Element | null)[]>([]);
    const lastDepsRef = useRef<DependencyList>([]);

    const unLoadRef = useRef<any>();

    useEffectType(() => {
      const targets = Array.isArray(target) ? target : [target];
      const els = targets.map((item) => getTargetElement(item));

      // init run
      if (!hasInitRef.current) {
        hasInitRef.current = true;
        lastElementRef.current = els;
        lastDepsRef.current = deps;

        unLoadRef.current = effect();
        return;
      }

      if (
        els.length !== lastElementRef.current.length ||
        !depsAreSame(els, lastElementRef.current) ||
        !depsAreSame(deps, lastDepsRef.current)
      ) {
        unLoadRef.current?.();

        lastElementRef.current = els;
        lastDepsRef.current = deps;
        unLoadRef.current = effect();
      }
    });

    useUnmount(() => {
      unLoadRef.current?.();
      // for react-refresh
      hasInitRef.current = false;
    });
  };

  return useEffectWithTarget;
};
```
