---
title: 内置对象
---

### WeakMap

- 弱类型的 Map，更有效的垃圾回收、释放内存。其内部的键作为一个弱键，当键被设置为 null，相关的值的数据也不在。
- [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)

测试代码：

```js
let a = { b: 123 };
let b = { a: 321 };

const c = new WeakMap();
c.set(a, a.b);

const d = new Map();
d.set(b, b.a);

console.log(c); //WeakMap {{…} => 123}

console.log(d); //Map(1) {{…} => 321}

a = null;
b = null;

//进行垃圾回收后
console.log(c); //WeakMap {}[[Entries]]No properties[[Prototype]]: WeakMap
console.log(d); //Map(1) {{…} => 321}
```

### Proxy：

- 创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。
- Object.defineProperty 和 Proxy 本质差别是，defineProperty 只能对属性进行劫持，所以出现了需要递归遍历，新增属性需要手动 Observe 的问题。
- 是新标准。
- [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

测试代码：例子中，当对象中不存在属性名时，默认返回值为 37。

```js
const handler = {
  get: function (obj, prop) {
    return prop in obj ? obj[prop] : 37;
  },
};

const p = new Proxy({}, handler);
p.a = 1;
p.b = undefined;

console.log(p.a, p.b); // 1, undefined
console.log('c' in p, p.c); // false, 37
```

### Reflect

- 提供拦截 JavaScript 操作的方法。
- 只有静态属性和方法，无法 new。
- 大部分与 Object 对象提供的方法相同。
- [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect)

以下方法为部分不同之处：

1. Reflect.set(target, propertyKey, value[, receiver])
   - 将值分配给属性的函数。返回一个 Boolean，如果更新成功，则返回 true。
2. Reflect.setPrototypeOf(target, prototype)
   - 设置对象原型的函数. 返回一个 Boolean， 如果更新成功，则返回 true。
