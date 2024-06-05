
---
title: Rxjs 高阶Map操作符
author: ICE
pubDatetime: 2024-06-5T16:55:12.000+00:00
slug: rxjs-high-level-operations
featured: false
draft: false
tags:
  - Rxjs
  - Js
description: "rxjs的高阶操作流"
---

在 RxJS 中，`ExhaustMap`、`MergeMap`、`SwitchMap` 和 `ConcatMap` 是四种常用的操作符，用于处理 Observable 流和进行数据转换。它们之间的区别在于它们在处理新的 Observable 时的行为不同。以下是它们的区别：

1. **`ExhaustMap`**：
    
    - 当源 Observable 发出值时，`ExhaustMap` 会将这个值映射成一个新的 Observable，并且只会订阅这个新 Observable。
    - 如果在这个新 Observable 订阅期间，源 Observable 再次发出值，那么这个值会被忽略，直到新 Observable 完成。如果在这个新 Observable 订阅期间，源 Observable 再次发出值，那么这个值会被忽略，直到新 Observable 完成。
    - 适用于处理顺序执行的任务，确保每个任务都执行完毕后再执行下一个任务。
    -  **在我们点击按钮发送异步的时候一直点击只会处理当前的，后面的会忽略掉 **
2. **`MergeMap`**：
    
    - 当源 Observable 发出值时，`MergeMap` 会将这个值映射成一个新的 Observable，并且会同时订阅所有这些新 Observable。
    - 新 Observable 可以并发执行，结果会按照它们完成的顺序合并。
    - 适用于并发执行多个任务，不需要等待前一个任务完成。
    - **主要用户创建新的流，一般用来处理多个请求 比如一个请求过来，拆分10个请求，每个请求再去请求100个数据拼接起来**。
1. **`SwitchMap`**：
    
    - 当源 Observable 发出值时，`SwitchMap` 会将这个值映射成一个新的 Observable，并且会取消订阅之前的 Observable，然后订阅新的 Observable。
    - 适用于处理只关心最新数据的情况，比如搜索建议等。
    - **主要就是可以忽略上一个请求，做竞态处理很好**
1. **`ConcatMap`**：
    
    - 当源 Observable 发出值时，`ConcatMap` 会将这个值映射成一个新的 Observable，并且会按顺序订阅这些新 Observable。
    - 新 Observable 会按照它们被订阅的顺序依次执行，一个完成后才会订阅下一个。
    - 适用于需要按顺序执行任务的情况，确保任务按照顺序执行。
    - **确保顺序执行**

总结：

- `ExhaustMap` 会忽略新值，直到当前 Observable 完成。
- `MergeMap` 会并发处理所有新值。
- `SwitchMap` 会取消订阅之前的 Observable，只关注最新的值。
- `ConcatMap` 会按顺序处理所有新值。

