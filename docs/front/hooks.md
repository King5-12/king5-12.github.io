---
title: ahooks理解
---

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
