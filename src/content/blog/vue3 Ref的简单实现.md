---
title: vue3 Ref的简单实现
author: ICE
pubDatetime: 2024-03-27T16:55:12.000+00:00
slug: vue3-Ref-simple-implement
featured: false
draft: false
tags:
  - Vue
  - Js
description: "Vue3 Ref简单实现"
---

### Ref介绍

`ref` 函数是 Vue 3 中用于创建响应式数据的一个函数。它接受一个初始值作为参数，并返回一个响应式引用对象。这个引用对象具有一个名为 `value` 的属性，用于存储和访问响应式数据的值。通过 `ref` 函数创建的响应式数据，当数据发生变化时，Vue 会自动追踪并更新。

### Ref使用方式

我们用单测的方式来看下ref的一些功能然后按照功能来一步一步实现Ref

```JavaScript
  it("happy path", () => {
    const a = ref(1);
    expect(a.value).toBe(1);
  });
```

1\首先需要传递一个值然后通过.value来返回它

```typescript
Class RefImpl{
	private _value: any

	constractor(val){
		this._value=value;
	}
	get value(){
		return this._value;
	}
	set value(value){
		this._value=value;
	}
}
export const function Ref(val){
	return RefImpl(val)
}
```

2、以上很简单接下来要实现响应，先看下单测

```javascript
it("should be reactive", () => {
  const a = ref(1);
  let dummy = a;
  let calls = 0;
  effect(() => {
    calls++;
    dummy = a.value;
  }); //初始化情况下都是1
  expect(calls).toBe(1);
  expect(dummy).toBe(1); //当value被set之后触发收集来的ff执行
  a.value = 2;
  expect(calls).toBe(2);
  expect(dummy).toBe(2); //再次设值因为值没有变所以effect方法不执行所以calls还是应该是2
  a.value = 2;
  expect(calls).toBe(2);
  expect(dummy).toBe(2);
});
```

先要创建一个dep来收集effect的方法，在get的时候收集 然后set的时候判断newvalue与oldvalue 是否相等，如果相等不做处理，不等就把newvalue赋值给\_value.然后触发执行dep里收集的方法。

```js
Class RefImpl{
	private _value: any
	public dep;

	constractor(val){
		this._value=value;
		this.dep=new Set();//防止多次添加同样的回调
	}
	get value(){
		//收集依赖
		trackRefValue(this)
		return this._value;
	}
	set value(newValue){
		if(Object.is(newValue,this._value)){
			this._value=value;
			triggerEffects(this.dep);//触发收集的依赖
		}
	}
}
export const function Ref(val){
	return RefImpl(val)
}

function trackRefValue(ref) {
  if (isTracking()) {
    trackEffects(ref.dep);
  }
}

```

关于收集的方法会在effect里介绍

### effect

传入一个函数被收集简单代码如下

```js

// 当前正在执行的 ReactiveEffect
let activeEffect = null;

class ReactiveEffect {
  private _fn: any;
  constructor(fn) {
    this._fn = fn;
  }
  // 执行 ReactiveEffect
  run() {
    activeEffect = this;
    this.fn();
  }
}

export const trackEffects = (dep) => {
  dep.add(activeEffect);
};

// 创建 effect 函数用于包裹 ReactiveEffect
function effect(fn) {
  const reactiveEffect = new ReactiveEffect(fn);
  reactiveEffect.run(); // 初始执行一次
  return reactiveEffect;
}

export function isTracking() {
  return  activeEffect !== undefined;
}

export const triggerEffects = (dep) => {
  for (const effect of dep) {
    if (effect.scheduler) {
      effect.run();
	 }

};

```

以上只是最简单的effects实现，实际情况要复杂点比如要传入options
是否要追踪shouldTrack、scheduler等方法。
