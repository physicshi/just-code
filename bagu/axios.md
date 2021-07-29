- [axios](#axios)
  - [拦截器原理](#拦截器原理)
  - [取消请求原理](#取消请求原理)

## axios

`axios`是一个基于`promise`的`HTTP`库，可以用在浏览器和`node.js`中。

在源码中主要是通过判断宿主环境是否存在`XMLHttpRequest`决定是使用浏览器的`XMLHttpRequest`还是`nodejs`的`http`模块。

### 拦截器原理

拦截就是在请求前或者响应后做操作，比如可以对请求的参数`config`做一些处理（比如统一添加cookie、设置请求头等）；或者得到响应数据后，对数据统一处理或者判断登陆态等。

拦截器部分本质上就是将请求拦截器，和响应拦截器，以及实际的请求（`dispatchRequest`）的方法组合成数组，类似如下的结构：

`[请求拦截器1success, 请求拦截器1error, dispatchRequest, undefined, 响应拦截器1success, 响应拦截器1error]`

> 一对对的是因为在`Promise`中，需要一个`success`和一个`fail`的回调函数，这个从代码`promise = promise.then(chain.shift(), chain.shift());`就能够看出来。因此，`dispatchReqeust`和`undefined`可以成为一对函数。

在`chain`执行队列中，发送请求的函数`dispatchReqeust`是处于中间的位置。它的前面是请求拦截器，通过`unshift`方法放入；它的后面是响应拦截器，通过`push`放入。

所以可以利用`promise.then()`的特点，自动通过`promise`链去调用方法。如果请求拦截器中有报错，则最后不会调`dispatchRequest`方法并断掉（因为`reject`的逻辑是`undefined`了）。

### 取消请求原理

> 核心是在`axios`源码中实现了一个`CancelToken`类（取消请求时，会挂载到具体的请求的`config`中），内部会创建一个`pending`状态的`promise`。
> 
> `CancelToken`类上会挂载`source`函数，返回`CancelToken`实例以及`cancel`函数（`CancelToken`实例参数传给的）
> 
> 当调用`CancelToken.source`暴露的`cancel`函数时，实例A中的`promise`状态由`pending`变成了`fulfilled`，立刻触发了`then`的回调函数，从而触发了`axios`的取消逻辑——`request.abort()`。

```js
// /cancel/CancelToken.js  -  11行
function CancelToken(executor) {
 
  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });
  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      return;
    }
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}
CancelToken.source = function source() {
    var cancel;
    var token = new CancelToken(function executor(c) {
        cancel = c;
    });
    return {
        token: token,
        cancel: cancel
    };
};


// /lib/adapters/xhr.js  -  159行
request.open()
// ...省略

if (config.cancelToken) {
    config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
            return;
        }
        // 取消逻辑
        request.abort();
        // 将axios返回的promise，置为reject态
        reject(cancel);
        // 重置请求
        request = null;
    });
}

// ...省略
request.send()
```

在可能需要取消的请求中，我们初始化时调用了`source`方法，这个方法内部返回了一个`CancelToken`类的实例A和一个函数`cancel`。

在`source`方法返回实例A中，初始化了一个在`pending`状态的`promise`。我们将整个实例A传递给`axios`实例后，这个`promise`被用于做取消请求的触发器。

当`source`方法返回的`cancel`方法被调用时，实例A中的`promise`状态由`pending`变成了`fulfilled`，立刻触发了`then`的回调函数，从而触发了`axios`的取消逻辑——`request.abort()`。

使用方法：

```js
import axios from 'axios'

// 第一种取消方法
axios.get(url, {
  cancelToken: new axios.CancelToken(cancel => {
    if (/* 取消条件 */) {
      cancel('取消日志');
    }
  })
});

// 第二种取消方法
const CancelToken = axios.CancelToken;
const source = CancelToken.source();
axios.get(url, {
  cancelToken: source.token
});
source.cancel('取消日志');
```


