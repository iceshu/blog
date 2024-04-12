---
title: vue proxy响应式
author: ICE
pubDatetime: 2024-03-27T16:55:12.000+00:00
slug: vue-simple-implement
featured: false
draft: false
tags:
  - Vue
  - Js
description: "Vue proxy 响应式"
---

### 一、最基础的响应

我们知道vue3里面的响应式非常好用，下面我将一步一步实现一个reactive。
1、实现效果

```typescript
let product = { price: 5, quantity: 2 };
let total = product.price * product.quantity; // 10 right?

product.price = 20;
console.log(`total is ${total}`);
```

我们想要当product的price 更改为20的时候自动触发

```typescript
let total = product.price * product.quantity; // 10 right?
```

我们先用最简单方法来处理更新的逻辑

```typescript

let product = { price: 5, quantity: 2 } ;
let total = 0 ;
let effect = function () {
total = product.price * product.quantity
});


effect() // 当product 修改的时候我们执行effect 就可以更新total

```

如何在需要更新的时候触发effect 就是我们下面要做的我们来创建一个Set结构的数组用来存放需要更新的effect。用set数据结构来确保effect的唯一性。

```javascript
let dep = new Set(); // 用来收集effect

function track() {
  dep.add(effect); //存取当前的effect
}

function trigger() {
  //用来执行收集的effect
  dep.forEach(effect => effect());
}
```

之后我们期望的代码执行应该如下(最基本的更新逻辑)

```typescript
let product = { price: 5, quantity: 2 };
let total = 0;
let dep = new Set();

function track() {
  dep.add(effect);
}

let effect = () => {
  total = product.price * product.quantity;
};

function trigger() {
  dep.forEach(effect => effect());
}

track();
effect();

product.price = 20;
console.log(total); // => 10

trigger();
console.log(total); // => 40
```

2、我们对有很多个属性，如何去存储这些set呢？
这里我们可以新建一个depsMap的Map对象来存储。

```typescript
const depsMap = new Map();
function track(key) {
  let dep = depsMap.get(key);
  if (!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  dep.add(effect);
}

function trigger(key) {
  let dep = depsMap.get(key);
  if (dep) {
    dep.forEach(effect => {
      effect();
    });
  }
}

let product = { price: 5, quantity: 2 };
let total = 0;

let effect = () => {
  total = product.price * product.quantity;
};

track("quantity");
effect();
console.log(total); // --> 10

product.quantity = 3;
trigger("quantity");
console.log(total); // --> 40
```

3、如果是多个对象的话此时就需要一个weakMap对象来做整体的存储代码如下
weakMap在删除key的时候可以释放对象。所以这里更加合适

```typescript
const targetMap = new WeakMap();
function track(target, key) {
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map())); // Create one
  }

  let dep = depsMap.get(key);

  if (!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  dep.add(effect);
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) {
    return;
  }
  let dep = depsMap.get(key);
  if (dep) {
    dep.forEach(effect => {
      effect();
    });
  }
}

let product = { price: 5, quantity: 2 };
let total = 0;
let effect = () => {
  total = product.price * product.quantity;
};

track(product, "quantity");
effect();
console.log(total); // --> 10
product.quantity = 3;

trigger(product, "quantity");
console.log(total); // --> 15
```

到此我们的代码初略的实现了依赖的收集，以及触发。现在是手动模式下面我们用proxy来优化下代码，以便当set属性的时候自动完成effect的运行

### 利用Proxy来自动追踪响应

1、在es6中我们可以用Proxy来拦截监听处理一个对象，首先我们在对象get的时候来收集，之后在对象set的时候来触发收集来的方法即可。

```typescript
const targetMap = new WeakMap(); // 收集存储effects然后每个对象更新的时候都要掉用收集来的effects

function track(target, key) {
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map())); // 如果没有就创建一个
  }
  let dep = depsMap.get(key); // 获取到对应key的dep
  if (!dep) {
    depsMap.set(key, (dep = new Set())); // Create a new Set
  }
  dep.add(effect); //将effect收集到dep里
}

function trigger(target, key) {
  const depsMap = targetMap.get(target); // 判断这个对象是否有一些key已经被依赖了
  if (!depsMap) {
    return;
  }
  let dep = depsMap.get(key);
  if (dep) {
    dep.forEach(effect => {
      effect();
    });
  }
}

function reactive(target) {
  const handler = {
    get(target, key, receiver) {
      let result = Reflect.get(target, key, receiver);
      track(target, key); // 依赖收集
      return result;
    },
    set(target, key, value, receiver) {
      let oldValue = target[key];
      let result = Reflect.set(target, key, value, receiver);
      if (result && oldValue != value) {
        trigger(target, key); // 触发依赖
      }
      return result;
    },
  };
  return new Proxy(target, handler);
}

let product = reactive({ price: 5, quantity: 2 });
let total = 0;
let effect = () => {
  total = product.price * product.quantity;
};
effect();

console.log("before updated quantity total = " + total);

product.quantity = 3;

console.log("after updated quantity total = " + total);
```

至此最简单的依赖收集+update更新出发更新实现
