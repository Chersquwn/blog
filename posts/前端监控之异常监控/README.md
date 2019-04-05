# 前端监控之异常监控

## 前言

随着业务的增长和团队的扩大，前端的应用也会越来越复杂，这时候只通过测试也很难保证能够覆盖所有的应用场景，而到线上等到用户集体反馈到客服，客服再反馈给开发，再去排查问题的时候，可能已经造成了严重的损失，这时候我们肯定会想，如果我能提前知道用户存在这些问题就好了，所以我们就需要一个异常监控系统，通过监控系统主动告知我们应用出现了问题。

一个完整的监控系统，大体上可以分为4个阶段：

- 日志采集
- 日志储存
- 日志统计和分析
- 日志报告和告警

本文将从日志采集这个阶段展开。

## 异常分类

在前端常见的异常有以下几种：

- 资源加载异常（包括css、js、图片等资源加载失败）
- 样式异常
- js运行时异常
- 接口异常

## 异常捕获

### 资源加载异常

资源加载异常通常可以分为两类，http劫持导致的资源异常和资源获取失败的异常。

- http劫持：

  http劫持通常分为三个小类：注入js劫持，iframe劫持和篡改html页面劫持，如何捕获这类异常在此不做展开，防御方法是直接上https。

- 资源获取失败：

  通过`window.addEventListener`监听error事件可以捕获资源加载异常, `ErrorEvent`中包含了详细的错误信息，网络请求异常不会冒泡，因此必须在capture阶段捕获。

  ```js
  window.addEventListener('error', event => {
    console.log(event)
  }, true)
  ```

### 样式异常

样式异常的捕获比较难以处理，单纯从前端代码中难以判断页面样式是否出现了异常，样式异常通常通过判断css是否加载失败，是否被劫持来分析。

### js运行时异常

js代码在运行过程中发生的异常，同样可以使用`window.addEventListener`监听error事件来捕获，为了能够捕获跨域js的异常，对应资源的标签上需要添加`crossorigin`属性。

```js
window.addEventListener('error', event => {
  const {
    filename,             // 发生错误的文件
    colno,                // 发生错误的列号
    lineno,               // 发生错误的行号
    message,              // 错误信息（字符串）
    error: { stack }      // 错误对象的堆栈信息
  } = event
  console.log(event)
}, false)
```

但是对于Promise的异常，通过`error`事件就无法捕获了，可以通过Promise的catch来捕获。

```js
new Promise((resolve, reject) => {
  throw 'Uncaught Exception!'
}).catch(error => {
  console.log(error)
});
```

如果未通过局部catch捕获的异常，这时候需要通过`unhandledrejection`事件来进行全局捕获。

```js
window.addEventListener('unhandledrejection', event => {
  const {
    reason     // 错误的原因，reject时的值
  } = event
  console.log(event)
}, false)
```

### 接口异常

对于接口异常的捕获，如果是针对某些http请求库，如axios，可以通过http请求库提供的拦截器来进行异常拦截，如果是要更为通用的拦截，通常是通过AOP的方式，对`XMLHttpRequest`进行函数劫持，http请求库的拦截器，基本也是这样实现的。

```js
const originOpen = XMLHttpRequest.prototype.open
const originSend = XMLHttpRequest.prototype.send

XMLHttpRequest.prototype.open = function open(method, url, bool) {
    originOpen.apply(this, [method, url, bool])
    // 这里可以进行一些处理
    this._url = url       // 记录当前接口的url
}

XMLHttpRequest.prototype.send = function send(_args) {
  this._startTime = Date.now()
  this.addEventListener('readystatechange', () => {
    if (this.readyState === 4) {
      this.timestamp = Date.now() - this._startTime     // 记录接口耗时

      if (this.status !== 200 && this.status !== 304 && this._url !== REPORT_URL) {
        // 这里可以进行异常上报
        console.log(`${this._url}接口发生异常`)
      }
    }
  }, false)

  originSend.apply(this, [_args])
}
```

## 结语

本文主要分析异常类型和对应异常的捕获，主要提供一个监控SDK捕获异常的部分设计思路。
