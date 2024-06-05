---
title: node 导出大量数据
author: ICE
pubDatetime: 2024-03-5T16:55:12.000+00:00
slug: node-export-huge-data
featured: false
draft: false
tags:
  - Node
  - Rxjs
  - Stream
description: "node 中间层导出大量excel数据，优化内存"
---

一般情况下node做中间层导出大致逻辑如下

![image.png](https://img.luqidong.com/obsidina20240119162121.png?v=1)

大致逻辑是这么一个流程：拿到前端请求---循环获取数据--处理数据---转换数据--写入文件--上传文件

1、其中“循环处获取数据“可以是获取第三方的也可以是获取数据库的，获取到的数据也许并不能直接使用所以要通过处理，这里就需要处理数据这么一个过程。比如数据里只有userid但是excel里需要的是username，那么在处理数据这边就需要拿到这一次所有的userid去请求得到username数据最后拼接好数据给下面逻辑处理，这里比较重要的是循环逻辑+错误重试。

2、其次是生成excel的过程，之前网上流行的库是xlsx ，用它做导出发现一个问题数据都是整个生成到xlsx的创建excel方法里导致内存暴涨，并没有用流的方式去写。后来自己手动用stream的方式去写数十万的数据内存几乎没变化。由于csv是文件所以可以直接自己写文件 核心代码如下

```typescript
const fileWriteStream = createWriteStream(filePath, {
  flags: "w", // 写流不能用r，会报错.可以用'a'表示追加
  autoClose: true, // 写完是否自动关闭
  encoding: "utf-8",
});
const headerRow = header.join(","); //第一行的数据

fileWriteStream.write(`\ufeff${headerRow}\n`); //写入第一行 这里前面为什么加ufeff 加了这个就一boom 用excel软件打开乱码 否则会有乱码

list.forEach(item => {
  //循环一行一行写入数据
  const values = header.map(e => item[e]).join(",");
  fileWriteStream.write(`${values}\n`);
});
fileWriteStream.end(); //文件写结束
```

3、后来发现网上有封装好的fast-csv就直接用它了 用法也是一样核心代码如下 他有个好处就是不需要手动设置header

```typescript
const csvStream = format({ headers: true, writeBOM: true }); //这里writeBom的功能就是我上面代代码加的ufeff
const filePath = "test.csv";
const fileWriteStream = createWriteStream(filePath, {
  flags: "w",
  autoClose: true, // 写完是否自动关闭
  encoding: "utf-8",
});
csvStream.pipe(fileWriteStream);
//写入数据
list.forEach(item => {
  csvStream.write(item);
});
csvStream.end(); //写入结束
```

4、整体流程伪代码如下 如果换成xslx 库来导出要用 Large Datasets 里的 XLSX.stream.to_csv方法 具体使用请查看文档 [https://docs.sheetjs.com/docs/demos/bigdata/stream/](https://docs.sheetjs.com/docs/demos/bigdata/stream/)

```typescript
  exportDataToCsv(data:any) {
    const {
      query,
    } = data;


    const csvStream = format({ headers: true, writeBOM: true });
    const filePath = resolve(
      __dirname,
      `../../../tempCsv/${downloadConfig.task.name}`,
    );
    const fileWriteStream = createWriteStream(filePath, {
      flags: 'w',
      autoClose: true, // 写完是否自动关闭
      encoding: 'utf-8',
    });
    csvStream.pipe(fileWriteStream);

    const requestFn = (
      query:any,
    ) => {
      return this.handleRequest(query); //这里处理了重试跟报错自己
    };
    let page = 1;
    const response$ = requestFn({...query,page});
    return response$.pipe( //循环每一次请求
      expand((response) => {
        if (response?.data.length === 0) {
          return EMPTY;
        }
        page++;
        return requestFn(
          { ...query,page},
          page,
        );
      }),
      takeWhile(Boolean), //如果获取不到数据了 则表示拉到了所有的数据
      map((response) => {  //这里拿到所有的数据如果这里的数据不能满足导出就要继续处理，我这里的例子是满足数据可以直接导出
        if (response.data) {
          return handleTransformData(response.data);
        }
        return throwError(() => new Error(response));
      }),
      map((list) => {
        list.forEach((item) => {
          csvStream.write(item);
        });
      }),
      toArray(), //集合所有的数据
      mergeMap( //这里比较重要必须要等文件流都写结束，否则待会上传就会缺少数据
        () =>
          new Promise((resolve) => {
            csvStream.end();
            fileWriteStream.on('finish', resolve);
          }),
      ),
      mergeMap(() => {
        return this.fileToButter(filePath); //我这里必须要转换成buff上传 其他的按照自己的上传逻辑
      }),
      mergeMap((buffer) => {
        console.log(`开始上传{filePath}`);
        return this.handleUploadData(buffer).pipe(
          map((e) => {
            return e.data;
          }),
        );
      }),
      catchError((e) => {
        this.logger.error(e, '导出逻辑报错');
      }),
      finalize(() => {
        console.log(`开始删除文件${filePath}`);
        unlinkSync(filePath);//删除文件
      }),
    );
  }
```
