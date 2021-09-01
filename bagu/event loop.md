- [事件循环](#事件循环)
  - [第一题](#第一题)
  - [第二题](#第二题)
  - [第三题](#第三题)

## 事件循环

### 第一题

```js
console.log("script start");

setTimeout(() => {
  console.log("setTimeout");
}, 1000);

Promise.resolve()
  .then(function () {
    console.log("promise1");
  })
  .then(function () {
    console.log("promise2");
  });

async function errorFunc() {
  try {
    await Promise.reject("error!!!");
  } catch (e) {
    console.log("error caught"); 
  }
  console.log("errorFunc");
  // 注意
  return Promise.resolve("errorFunc success");
}
errorFunc().then((res) => console.log("errorFunc then res"));

console.log("script end");
```

输出顺序是什么？

要注意，异步的任务注册后再把异步的结果和回调函数放到任务队列。对于 async await，**await 会阻塞异步操作**, 所以这个 await 后面的 Promise 不会去回调队列排队，而是等待完成：

```js
script start
script end
promise1
error caught
errorFunc
promise2
errorFunc then res
setTimeout
```

**对于 node 中的[事件循环](https://juejin.cn/post/6844904079353708557#heading-2)，主要分为 6 个阶段，node11 版本一旦执行一个阶段里的一个宏任务(setTimeout,setInterval 和 setImmediate)就立刻执行对应的微任务队列。（这一点和浏览器中是一致的）**。

### 第二题

```js
console.log("script start");

async function async1() {
  await async2();
  console.log("async1 end");
}
// 这里async2其实是一个同步函数
async function async2() {
  console.log("async2 end");
}
async1();

setTimeout(function () {
  console.log("setTimeout");
}, 0);

new Promise((resolve) => {
  console.log("Promise");
  resolve();
})
  .then(function () {
    console.log("promise1");
  })
  .then(function () {
    console.log("promise2");
  });
```

执行顺序为：

script start -> async2 end -> Promise -> async1 end -> promise1 -> promise2

### 第三题

来看一下这个结果：

```js
console.log("script start");

async function async1() {
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2 end");
  // 这里 resolve 会注册一个微任务，然后去执行async外面的任务
  return Promise.resolve().then(() => {
    console.log("async2 end1");
  });
}
async1();

setTimeout(function () {
  console.log("setTimeout");
}, 0);

new Promise((resolve) => {
  console.log("Promise");
  resolve();
})
  .then(function () {
    console.log("promise1");
  })
  .then(function () {
    console.log("promise2");
  });

console.log("script end");
```

script start ->async2 end -> promise ->script end -> async2 end1 ->promise1 -> promise2 ->async1 end ->setTimeout

> async 函数会返回一个 Promise 对象，如果在函数中 return 一个直接量（普通变量），async 会把这个直接量通过 Promise.resolve() 封装成 Promise 对象。如果你返回了 promise 那就以你返回的 promise 为准。没有return就是返回了Promise.resolve(undefined)

这里可能 async1 里面的逻辑有一丝混乱，**如果 await 后面直接跟的为一个变量，比如：await 1；这种情况的话相当于直接把 await 下一行的代码注册为一个微任务，可以简单理解为 promise.resolve(1).then(await 下面的代码)**。

在本题中，如果 await 后面跟的是一个异步函数的调用，await 下一行的代码会在本轮的最后执行。

```js
//resolve方法
Promise.resolve = function(val) {
    return new Promise((resolve,reject)=>{
        if(val instanceof Promise){
            val.then(val => {
                resolve(val);
            }, err => {
                reject(err);
            });
        }else{
            resolve(val);
        }
    }) 
}
```

`Promise.resolve()`如果参数是`promise`实例，还会多加一层（会有两个`promise`）：`promise.resolve().then(()=>{console.log()}).then(()=>{resolve()}).then(()=>{console.log()})`

`promise.resolve().then();promise.resolve().then()`

> ~~此时执行完 await 并不先把 await 下一行的代码注册到微任务队列中去，而是执行完 await 之后，直接跳出 async1 函数，执行其他代码。然后遇到 promise 的时候，把 promise.then 注册为微任务。其他代码执行完毕后，需要回到 async1 函数把 await 下一行的代码注册到微任务队列当中，注意此时微任务队列中是有之前注册的微任务的。所以这种情况会先执行 async1 函数之外的微任务(promise1,promise2)，然后才执行 async1 内注册的微任务(async1 end). 可以理解为，这种情况下，await 下一行的代码会在本轮循环的最后被执行。~~

如果上面那个例子没有`return`：

```js
console.log("script start");

async function async1() {
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2 end");
  Promise.resolve().then(() => {
    console.log("async2 end1");
  });
}
async1();

setTimeout(function () {
  console.log("setTimeout");
}, 0);

new Promise((resolve) => {
  console.log("Promise");
  resolve();
})
  .then(function () {
    console.log("promise1");
  })
  .then(function () {
    console.log("promise2");
  });

console.log("script end");
```

结果为：

```
script start
async2 end
Promise
script end
async2 end1
promise1
promise2
setTimeout
```

相当于：

```js
async function async2() {
  console.log("async2 end");
  // 这里 resolve 会注册一个微任务，然后去执行async外面的任务
  Promise.resolve().then(() => {
    console.log("async2 end1");
  });
// 所以这里会连续注册两个微任务
return Promise.resolve()
}
```
