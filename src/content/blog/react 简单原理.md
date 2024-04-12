---
title: React简单实现
author: ICE
pubDatetime: 2024-03-25T16:55:12.000+00:00
slug: react-simple-implement
featured: false
draft: false
tags:
  - React
  - Js
description: "简简单单实现一个react？"
---

一、React 中有虚拟dom，从虚拟dom到real dom这个过程
分析：虚拟dom是用js对象去描述真实dom节点的对象 demo如下

```typescript
const vnode = {
  type: "TEXT_ELEMENT",
  props: {
    id: "app",
  },
};
//那么创建text 节点封装方法就是
function createTextNode(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}
//其他节点就是
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child => {
        return typeof child === "string" ? createTextNode(child) : child;
      }),
    },
  };
}
```

1、react 是从vdom到real element

- 创建element 可以用document.createElement 如果是text的类型则是 document.createTextNode
- 设置属性通过props里面循环key来设置
- append到container上
- 继续处理children来递归处理

```typescript
function render(el, container) {
  const dom =
    el.type === "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(el.type);
  Object.keys(el.props).forEach(key => {
    if (key !== "children") {
      dom[key] = el.props[key];
    }
  });
  const children = el.props.children;
  children.forEach(child => {
    render(child, dom);
  });
  container.append(dom);
}
```

创建一个App.js

```typescript
const App = React.createElement("div", { id: "app" }, "hello world");
```

执行

```typescript
render(App, document.getElementById("root"));
```

不出意外的话就可以渲染处结果了

二、fiber的简单原理
当要渲染的children太多的时候就会容易卡顿，fiber思想就是当浏览器闲置的时候执行渲染操作从而减少页面卡顿的情况。
思路就是把渲染children的操作都放到任务队列里等闲置的时候去执行。vnode 是一个树的结构如何去组合这些子节点呢？可以通过链表的方式来串起来。最后只要判断浏览器闲置这里可以通过浏览器自带的·requestldleCallback·来做任务的分割。文档地址https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback

1、实现树结构转成链表

- 找子节点
- 找兄弟节点
- 找兄弟父节点
  2、requestldleCallback来依次执行链表

3、所以要来实现一个生成链表的操作

```typescript
function transformToChain(fiber) {
  const children = fiber.props.children;
  let prevChild = null; //这里很巧妙 设置一个变量来存上一次的数据然后好指定sibling为新的newFiber
  children.forEach((child, index) => {
    const newFiber = {
      type: child.type,
      props: child.props,
      child: null,
      parent: fiber,
      sibling: null,
      dom: null,
    };
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevChild.sibling = newFiber;
    }
    prevChild = newFiber;
  });
}
```

解释如下：
假设我们有一个父 fiber 节点 A，它有两个子节点 B 和 C，子节点 B 又有一个子节点 D。每个子节点都会被转换为一个新的 fiber 对象，并通过链表结构连接起来。

```text
            +-----+
            |  A  |
            +-----+
              |
   +----------+----------+
   |          |          |
+-----+     +-----+
|  B  |     |  C  |
+-----+     +-----+
   |
+-----+
|  D  |
+-----+


```

1. 首先，函数从父 fiber 节点 A 中获取子节点数组，假设子节点数组为 `[B, C]`。
2. 开始遍历子节点数组：
   - 对于第一个子节点 B，创建一个新的 fiber 对象 B'，并初始化其属性，然后将 B' 赋值给父节点 A 的 `child` 属性，此时链表结构为 `A -> B'`。
     - 对于子节点 B 的子节点 D，创建一个新的 fiber 对象 D'，并初始化其属性，然后将 D' 赋值给子节点 B' 的 `child` 属性，此时链表结构为 `A -> B' -> D'`。
   - 对于第二个子节点 C，创建一个新的 fiber 对象 C'，并初始化其属性，然后将 C' 赋值给前一个子节点 B 的 `sibling` 属性，此时链表结构为 `A -> B' -> D' -> C'`。
3. 循环结束后，链表结构就建立完成了，每个子节点都连接到了它的兄弟节点和子节点，形成了一个更加详细的链表结构，用于描述子节点之间的层级关系。
4. 这里为什么要先找child字节点？因为挂载渲染的时候是要dom的，必须要先找到子节点，没有的话找兄弟，之后找叔叔节点。
   4、创建一个循环执行

```javascript
function workLoop(deadline) {
  let shouldYield = false;
  while (!shouldYield && nextWorkOfUnit) {
    nextWorkOfUnit = performWorkOfUnit(nextWorkOfUnit);
    shouldYield = !deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}

function performWorkOfUnit(fiber) {
  if (!fiber.dom) {
    const dom = (fiber.dom = createDom(fiber.type));
    fiber.parent.dom.append(dom);
    updateProps(dom, fiber.props);
  }
  transformToChain(fiber);
  return fiber.child || fiber.sibling || fiber.parent?.sibling;
}
```

看起来很完美，但是问题fiber每一次空闲的时候插入dom，这样有可能不空闲的时候就会卡住dom不渲染了。所以我们在performWorkOfUnit的时候不需要做dom的插入，直接收集好dom在内存里，直到所有的dom都生成好 一次性插入。
这里就要坐下简单的修改 1、增加插入dom的一次性方法，2、增加递归插入child跟siblings插入结束就能够渲染了。

```javascript
function performWorkOfUnit(fiber) {
  if (!fiber.dom) {
    const dom = (fiber.dom = createDom(fiber.type)); //fiber.parent.dom.append(dom);
    updateProps(dom, fiber.props);
  }
  transformToChain(fiber);
  return fiber.child || fiber.sibling || fiber.parent?.sibling;
}

//创建一个递归的appenddom的方法

function commitWork(fiber) {
  fiber.parent.dom.append(fiber.dom);
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
//创建一个总提交方法 只执行一次
function commitRoot() {
  commitWork(root.child);
  root = null;
}

function workLoop(deadline) {
  let shouldYield = false;
  while (!shouldYield && nextWorkOfUnit) {
    nextWorkOfUnit = performWorkOfUnit(nextWorkOfUnit);
    shouldYield = !deadline.timeRemaining() < 1;
  }
  if (!nextWorkOfUnit && root !== null) {
    commitRoot();
  }
  requestIdleCallback(workLoop);
}
```

似乎这样就很完美了，但是好像有个问题~
当我们渲染这样的结构

```html
<div>
  <p>
    <span>1</span>
    <span>2</span>
  </p>
  <p>3</p>
</div>
```

发现p3是无法渲染的~因为当渲染到2的时候寻找父亲节点的sibling 是null 应该要继续寻找父节点的sibling 所以performWorkOfUnit的return 下一个节点的逻辑需要修改

```javascript
if (fiber.child) {
  return fiber.child;
}
let nextFiber = fiber;
while (nextFiber) {
  if (nextFiber.sibling) return nextFiber.sibling;
  nextFiber = nextFiber.parent;
}
```

真是走一步不看一步啊

### 集成function Component

渲染function component 其实就是一个拆箱的过程，由于function component 返回的是一个function 那么我们可以之直接判断 fiber.type的类型是function的话直接执行，然后返回的dom去渲染。

```javascaript
 const isFunctionComponent=typeof fiber.type ==='function'
```

这里需要注意的是由于function component 他不是一个真实的dom 我们在寻找dom挂载的时候需要一直往上找

```javascript
function commitWork(fiber) {
  if (!fiber) return;
  let fiberParent = fiber.parent;
  while (!fiberParent.dom) {
    fiberParent = fiberParent.parent;
  } //这里有可能是Counter 没有dom
  if (fiber.dom) {
    fiberParent.dom.append(fiber.dom);
  }
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

App.jsx

```javascript
import React from "./core/React.js";

function Counter({ num }) {
  return <div>counter:{num}</div>;
}

function App() {
  return (
    <div>
            111      {" "}
      <div>
                aaa <p id="p2">2</p>     {" "}
      </div>
            <p id="p">bbb</p>      <Counter num={12}></Counter>     {" "}
      <Counter num={14}></Counter>   {" "}
    </div>
  );
}
export default App;
```

### 组件的更新

1、组件更新我们先加一个事件，props里的属性我们已经知道怎么加了，事件就是onClik这样的类型 就是startWith ‘on’开头即可所以 在updateProps中代码需要增加这段逻辑

```javascript
function updateProps(dom, props) {
  Object.keys(props).forEach(key => {
    if (key.startsWith("on")) {
      const eventType = key.toLowerCase().substring(2);
      dom.addEventListener(eventType, props[key]);
    } else {
      if (key !== "children") {
        dom[key] = props[key];
      }
    }
  });
}
```

2、更新逻辑梳理，我们已经可以按到vnode对象并且已经链表起来了，所以update只是循环之前的vnode链表然后判断props是否改变从而更新。更新包含删除节点、更新节点、以及新增。现在只要拿到新的vnode与 旧的比较就行。

- 删除节点基本dom上面的remove方法就行
- 更新跟新增节点可以是一个逻辑
  - update props 逻辑可以分成如果老的里面有新的里没有就是直接删除
- 更新可以记录当前处理的fiber 这样不需要全局更新、如何拿到当前需要更新的节点可以在更新functionComponent里记录下来

### 新增useState方法

1、参照react的useState书写处整个useState的处理逻辑

```javascript
let stateHooks = null; //存放hooks 因为页面会有多个useState
let stateHooksIndex = 0; //当前处理的hook索引
function useState(initialState) {
  let currentFiber = wipFiber; //获取当前处理的fiber
  const oldHook = currentFiber?.alternate?.stateHooks[stateHooksIndex]; //找到老的fiber
  const stateHook = {
    state: oldHook?.state || initialState, //获取老的state
    queue: oldHook?.queue || [], //存放多个action
  };
  stateHook.queue.forEach(action => {
    stateHook.state = action(stateHook.state);
  });
  stateHook.queue = [];
  stateHooksIndex++;
  stateHooks.push(stateHook);
  currentFiber.stateHooks = stateHooks;
  function setState(action) {
    const fn = typeof action === "function" ? action : () => action;
    const eagerState = fn(stateHook.state);
    if (eagerState === stateHook.state) return; //比较值是否相等
    stateHook.queue.push(fn);
    wipRoot = {
      ...currentFiber,
      alternate: currentFiber,
    };
    nextWorkOfUnit = wipRoot;
  }

  return [stateHook.state, setState];
}
```

2、梳理

- 因为一个functionComponent里会有多个useState方法调用所以要有个变量存下来
- 执行setState方法可以先判断新值与老的是否相等
- 1. **初始化变量**：
  - `stateHooks`: 用于存放所有的状态钩子对象。
  - `stateHooksIndex`: 用于跟踪处理的状态钩子的索引。

2. **useState 函数**：

   - 当调用 `useState(initialState)` 时，会创建一个状态钩子对象 `stateHook`，其中包含状态值 `state` 和动作队列 `queue`。
   - 如果存在旧的状态钩子对象（`oldHook`），则从中获取先前的状态和动作队列。
   - 对动作队列中的每个动作执行，并更新状态值。
   - 清空动作队列。
   - 将新的状态钩子对象添加到 `stateHooks` 中，并更新当前处理的 fiber 的状态钩子。

3. **setState 函数**：

   - `setState` 函数用于更新状态值。
   - 接受一个动作函数 `action`，并将其添加到状态钩子对象的动作队列中。
   - 如果计算出的新状态值与当前状态值相等，则不进行更新。
   - 如果状态值有更新，将动作函数添加到队列中，并更新 `wipRoot`。

4. **返回值**：

   - `useState` 函数返回一个数组，包含当前状态值和更新状态的 `setState` 函数。

整体流程如下：

1. 调用 `useState(initialState)` 来创建一个状态钩子对象。
2. 使用返回的状态值和 `setState` 函数来管理和更新状态。
3. 每次状态更新都会创建一个新的状态钩子对象，并将其添加到状态钩子数组中。

### useEffect 简单实现

这就稍微简单一些

```javascript
function useEffect(callBack, deps) {
  const effectHook = {
    callBack,
    deps,
    cleanUp: undefined, //用来存储清楚effect的副作用比如removeEventLstener
  };
  effectHooks.push(effectHook);
  wipFiber.effectHooks = effectHooks;
}
```

useEffect在commitRoot方法里调用

```javascript
function commitEffectHooks() {
  function run(fiber) {
    if (!fiber) return;
    if (!fiber.alternate) {
      fiber?.effectHooks?.forEach(newHook => {
        newHook.cleanUp = newHook.callBack();
      });
    } else {
      fiber?.effectHooks?.forEach((newHook, index) => {
        if (newHook.deps.length === 0) return;
        const oldEffectHook = fiber.alternate.effectHooks[index];
        const needUpdate = oldEffectHook.deps.some((oldDep, i) => {
          return oldDep !== fiber.effectHook?.deps[i];
        });
        needUpdate && (newHook.cleanUp = newHook?.callBack());
      });
    }

    run(fiber.child);
    run(fiber.sibling);
  }

  function runCleanUp(fiber) {
    if (!fiber) return;
    fiber?.alternate?.effectHooks?.forEach(hook => {
      if (hook?.deps?.length) {
        hook?.cleanUp?.();
      }
    });
    runCleanUp(fiber.child);
    runCleanUp(fiber.sibling);
  }
  runCleanUp(wipRoot);
  run(wipRoot);
}
```

1. **run 函数**：

   - 该函数用于遍历 fiber 树中的每个节点，执行副作用钩子的回调函数并处理更新。
   - 如果当前 fiber 节点没有旧版本（`alternate`），则表示是新创建的节点，直接执行所有新的副作用钩子的回调函数，并将返回的清理函数（cleanUp）保存在 `cleanUp` 属性中。
   - 如果当前 fiber 节点有旧版本，则比较新旧副作用钩子的依赖数组，如果依赖有变化，则执行新的副作用钩子的回调函数，并将返回的清理函数保存在 `cleanUp` 属性中。

2. **runCleanUp 函数**：

   - 该函数用于遍历 fiber 树中的每个节点，执行需要清理的副作用钩子的清理函数（cleanUp）。
   - 对于有依赖的副作用钩子，如果依赖有变化，则执行清理函数。

3. **执行顺序**：

   - 首先调用 `runCleanUp(wipRoot)`，执行对副作用钩子的清理操作。
   - 然后调用 `run(wipRoot)`，执行对副作用钩子的更新操作。

4. **递归遍历**：

   - 在 `run` 和 `runCleanUp` 函数中，通过递归遍历 fiber 树的子节点和兄弟节点，确保对整棵 fiber 树中的每个节点都进行副作用钩子的处理。
