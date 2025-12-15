---
title: call 实现
date: 2025-12-12 18:00:00
categories:
  - 手写实现
  - js
tags:
  -
# slug:
# description:
# cover: /images/cover/my-first-post.jpg
---

## call、bind 和 apply 的介绍

三者的作用都很类似，都是改变函数的 this 指向
但是  ***call 和 apply 用来改变 this 并立即调用函数；bind 用来返回一个永久绑定 this 的新函数***。

实例：

{% hideToggle 展开 / 收起示例代码,#49b1f5,#fff %}

```js
function greet(greeting, punctuation) {
  console.log(greeting + ', ' + this.name + punctuation);
}

const person = { name: 'Alice' };
const anotherPerson = { name: 'Bob' };
```

call **改变 this 并立即调用**

```js
greet.call(person, 'Hello', '!');
// 输出：Hello, Alice!

```

apply **改变 this 并立即调用(调用参数不同)**

```js
greet.apply(person, ['Hi', '!!!']);
// 输出：Hi, Alice!!!

```

bind **改变 this，但不会立即调用 → 返回一个新函数**

```js
const greetBob = greet.bind(anotherPerson, 'Hey');
greetBob('?');
// 输出：Hey, Bob?

```

{% endhideToggle %}

## 手写call

接下来将会手写 `call` `apply` 和 `bind`

```js
Function.prototype.mycall = function (ctx, ...args) {
  // 保持key的唯一性
  const key = Symbol()
  // 确定this指向
  // globalThis 指向全局对象，因为宿主环境不同，指向不同。例如浏览器和node环境
  ctx = ctx === undefined || ctx === null ? globalThis : Object(ctx)
  ctx[key] = this
  var result = ctx[key](...args);
  delete ctx[key]

  return result
}
```

整体看下来其实很简单，就是将调用函数的this指向放到ctx里。接下来进行简单测试

```js
function fn(a, b) {
    // console.log(this);

    return a + b
}

const res1 = fn.mycall({}, 2, 3)
const res2 = fn.mycall(null, 2, 3)
const res3 = fn.mycall(undefined, 2, 3)
const res4 = fn.mycall(1, 2, 3)
console.log("res1", res1);
console.log("res2", res2);
console.log("res3", res3);
console.log("res4", res4);
```

下面是测试结果

```bash
res1 5
res2 5
res3 5
res4 5
```

## 手写apply

```js
Function.prototype.myapply = function (ctx, args) {
    const key = Symbol()
    ctx = ctx === undefined || ctx === null ? globalThis : Object(ctx)
    ctx[key] = this
    let result

    // 这里分出区别，如果args为空，则直接调用函数，没有则传入args
    if (args) {
        args = Array.prototype.slice.call(args)
        result = ctx[key](...args);
    } else {
        result = ctx[key]()
    }
    delete ctx[key]

    return result
}
```

`apply` 与 `call` 比较类似。主要区别就是传入参数的不同。从此区别来更改 `call` 的写法就可以实现

测试

```js
const res5 = fn.myapply({}, [2, 3])
const res6 = fn.myapply(null, [2, 3])
const res7 = fn.myapply(undefined, [2, 3])
const res8 = fn.myapply(1, [2, 3])
const applyempty = fn.myapply({}, [2, 3])
console.log("res5", res5);
console.log("res6", res6);
console.log("res7", res7);
console.log("res8", res8);
console.log("applyempty", applyempty);
```

结果为：

```bash
res5 5
res6 5
res7 5
res8 5
applyempty 5
```

## 手写bind

```js
Function.prototype.myBind = function (context) {
    // 获取参数
    // arguments 为传入的参数，值为传入的参数。注意：这里从 slice(1) 开始。因为这里第一个参数是context
    var args = [...arguments].slice(1),
        fn = this;
    return function Fn() {
        // 根据调用方式，传入不同绑定值
        return fn.myapply(
            this instanceof Fn ? this : context,
            args.concat(...arguments)
        );
    };
};

```
