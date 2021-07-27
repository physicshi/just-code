- [详解模块化](#详解模块化)
- [`Nodejs`的事件循环](#nodejs的事件循环)
- [`http`模块](#http模块)
  - [创建服务器](#创建服务器)
  - [主机和端口](#主机和端口)
  - [`request`对象](#request对象)
    - [`URL`的处理](#url的处理)
      - [路径的处理](#路径的处理)
    - [请求头](#请求头)
    - [响应](#响应)
      - [返回响应结果](#返回响应结果)
      - [状态码](#状态码)
      - [响应头](#响应头)
- [`express`框架](#express框架)
  - [hello express](#hello-express)
  - [App.use((req,res,next)=>{})](#appusereqresnext)
  - [中间件](#中间件)
- [`koa`框架](#koa框架)
  - [koa的中间件模型](#koa的中间件模型)

## 详解模块化



## `Nodejs`的事件循环



## `http`模块

当应用程序（客户端）需要某一个资源时，可以向一台服务器，通过`Http`请求获取到这个资源；提供资源的这个服务器，就是一个`Web`服务器；

<img src="./pic/webserver.png">

目前有很多开源的Web服务器：`Nginx`、`Apache（静态）`、`Apache Tomcat（静态、动态）`、`Node.js`

```js
const http=require("http");
const server=http.createServer((req,res)=>{
    res.end("hello server");
});

server.listen(8000,()=>{
    console.log("服务器启动成功")
})
```

需要注意的是，`server.listen`是一个异步方法，也许有人习惯这样写：

```js
server.listen(8000);
console.log("服务器启动成功");
```

但是这样不管启动成功还是失败，最终都会打印出`服务器启动成功`这段话，所以真正的监听要放在`server.listen`里面。

### 创建服务器

一般来讲创建服务器对象有两种方法：

+ 通过`http.createServer((req,res)=>{})`，直接返回服务器对象；
+ 也可以直接`new http.Server((req,res)=>{})`来创建这个对象；

来看一下源码：

```js
function createServer(opts, requestListener) {
  return new Server(opts, requestListener);
}
```

可以看到底层都是用的`new Server`对象；

### 主机和端口

`Server`通过`listen`方法来开启服务器，并且在某一个主机和端口上监听网络请求；就是当我们通过`ip:port`的方式发送到我们监听的Web服务器上时，就可以对其进行相关的处理；

```js
const server=http.createServer((req,res)=>{
    req.end("hello node")
})
server.listen(8888,"127.0.0.1",()=>{
    console.log("服务器启动成功")
})
```

`listen`方法有三个参数：

+ 端口`port`：可以不传，系统会默认分配端；

  > 可以在回调中利用`console.log(server.address().port)`拿到默认分配的端口号；

+ 主机host：通常可以传入`localhost`、`ip`地址127.0.0.1、或者`ip`地址`0.0.0.0`，默认是`0.0.0.0`；

  + `localhost`：本质上是一个域名，通常情况下会被解析成`127.0.0.1`；

  + `127.0.0.1`：回环地址（`Loop Back Address`），表达的意思其实是我们主机自己发出去的包，直接被自己接收；

    + 正常的数据库包经常应用层- 传输层- 网络层- 数据链路层- 物理层；

    + 而回环地址，是在网络层直接就被获取到了，是不会经过数据链路层和物理层的；

      > 比如我们监听127.0.0.1时，在同一个网段下的主机中，通过`ip`地址是不能访问的；

  + `0.0.0.0`：

    + 监听`IPV4`上所有的地址，再根据端口找到不同的应用程序；

      > 比如我们监听`0.0.0.0`时，在同一个网段下的主机中，通过`ip`地址是可以访问的；

+ 回调函数：服务器启动成功时的回调函数；

### `request`对象

Node会封装一个`request`对象，包含了客户端向服务器发送的所有信息。

#### `URL`的处理

`req.url`本质上是拿到形如`/login?a=1&b=2&c=3`，即**路径名加上`qurery`**

##### 路径的处理

+ `req.url`中只有简单的路径

```js
const http=require("http");
const server=http.createServer((req,res)=>{
    if(req.url==="/login"){
       	res.end("欢迎回来~")
       }else if(req.url==="/products"){
         res.end("商品页~")  
       }else{
           res.end("404")
       }
});
```

+ `qurery`的处理

在`Node.js`里提供了内置模块——`url`模块，用于处理`url`地址

`url.parse`用来解析地址。把想要解析的`req.url`传给`url.parse`，解析成对象。

```js
const http=require("http");
const url=require("url");
const server=http.createServer((req,res)=>{
    console.log(url.parse(req.url))
    res.end("hello server");
    
});

server.listen(8000,()=>{
    console.log("服务器启动成功!")
   
})
```

以`localhost:8000/login?a=1&b=3`为例，我们拿到的解析后的对象是：

```js
Url {
  protocol: null,
  slashes: null,
  auth: null,
  host: null,
  port: null,
  hostname: null,
  hash: null,
  search: '?a=1&b=3',
  query: 'a=1&b=3',
  pathname: '/login',
  path: '/login?a=1&b=3',
  href: '/login?a=1&b=3'
}
```

我们真正需要的就是`pathname`和`query`，需要另一个内置模块`querystring`对`query`部分相应的处理：

```js
const http=require("http");
const url=require("url");
const qs=require("querystring");

const server=http.createServer((req,res)=>{
    const {pathname,query}=url.parse(req.url);
    if(pathname==="/login"){
        const {a,b}=qs.parse(query);
        console.log(a,b);//1 3
    }
    res.end("hello server");
});

server.listen(8000,()=>{
    console.log("服务器启动成功!")
   
})
```

####　请求方法

我们可以通过`req.method`来判断请求方式。

> 背景是在`body`传了一个`JSON`字符串。

比如用`POST`请求，我们一般会把数据放到`body`中，我们没办法用形如`req.body`的方式拿到客户端传来的结果，可以用`req.on("data",(data)=>{console.log(data)})`的方式拿到，不过直接可以拿到的是`buffer`流，我们还需要用`req.on("data",(data)=>{console.log(data.toString())})`，或者在开始写好`req.setEncoding("utf-8")`——对应`body`里是文本；`req.setEncoding("binary")`——对应`body`里是图片或者音视频。

> 不过最后拿到的`data`是一个字符串类型，我们要用`JSON.parse()`将其转成对象，再解构或者做别的操作~

```js
const http=require("http");
const url=require("url");

const server=http.createServer((req,res)=>{
  const {pathname}=url.parse(req.url);
  if(pathname==="/login"){
     if(req.method==="POST"){
       req.setEncoding("utf-8");
       req.on("data",(data)=>{
         const {username , age}=JSON.parse(data);
         console.log(username ,age)/xiaoming 18
       })
       req.end("hello world")
     }
  }
})
server.listen(8888,"0.0.0.0",()=>{
  console.log("服务器启动成功～")
})
```

> 流是一种以高效的方式处理读/写文件、网络通信、或任何类型的端到端的信息交换。
>
> 在传统的方式中，当告诉程序读取文件时，这会将文件从头到尾读入内存，然后进行处理。使用流，则可以逐个片段地读取并处理（而无需全部保存在内存中）。
>
> 相对于使用其他的数据处理方法，流基本上提供了两个主要优点：
>
> - **内存效率**: 无需加载大量的数据到内存中即可进行处理。
> - **时间效率**: 当获得数据之后即可立即开始处理数据，这样所需的时间更少，而不必等到整个数据有效负载可用才开始。

#### 请求头

我们可以通过`req.headers`拿到请求头。

#### 响应

##### 返回响应结果

一般有两种方式来返回响应结果：

+ `write`方法：直接写数据，并没有关闭流；
  + `res.write("111")`
+ `end`方法：写出最后的数据，写完后会关闭流；
  + `res.end("message end")`

注意，如果没有调用`end`，客户端会一直等待返回结果；所以客户端在发送网络请求时，会设置超时时间

##### 状态码

返回状态码有两种方式：

+ `res.statusCode=400;`

+ `res.writeHead(200);`

##### 响应头

+ `res.setHeader()`：一次写入一个头部信息；
+ `res.writeHead()`：同时写入`header`和`status`；

```js
res.setHeader("Content-Type":"application/json;charset=utf8");

res.writeHead(200,{"Content-Type":"application/json;charset=utf8"})
```

## `express`框架

在正式介绍`express`框架前，我们先说一下`web`框架：

+ 更方便的处理`HTTP`请求和响应；
+ 更方便的连接数据库，`Redis`
+ 更方便的路由

理念：

+ `Web`框架的主流思路都是`MVC`

+ `Model`处理数据相关逻辑
+ `View`处理视图相关逻辑，前后分离之后，`View`不重要（都交给前端完成了）
+ `Controller`负责其他逻辑

在我看来，这就是`web`框架的全部了。

具体到`nodjs`，我们的web框架就是基于nodjs API的封装，我们的代码既可以基于`web`框架也可以基于其底层的`nodejsAPI`；不管是基于什么，最终的目的都是对浏览器发来的`http`请求作出响应；连接 `mysql`数据库，向`mysql`数据库发送`sql`语句，然后`mysql`服务器返回对应的数据；或者向`redis`服务器发送请求，然后`redis`服务器就会返回对应的数据结果；或者是向其他的`web`服务器发送请求，接收对应服务器的响应（比如爬虫）。

### hello express

```
git init
npm init
npm i express
```



```js
const express=require("express");
const app=express();

app.get("/",(req,res)=>{
    res.send("hello express");
})

const poort=8080;
app.listen(port,()=>{
    console.log(`监听${port}`)
})
```

加上`ts`：

```
npm i -g typescript ts-node ts-node-dev
npm i -D @types/express
tsc --init
```

修改`tsconfig.json`中 `target:"es2015", noImplicitAny:true,`，即改为`es6`以及没有隐式的`any`：

```js
import express from "express"
```

### App.use((req,res,next)=>{})

```js
import express from "express";
const app=express();

app.use((req,res,next)=>{
  res.write("hi")
  next()
})

app.use((req,res,next)=>{
  res.write("hi")
  next()
})

app.use((req,res,next)=>{
  res.end()
})

app.listen(3000,()=>{
  console.log("监听3000")
})
```

### 中间件

+ 中间件本质上就是传递给express的回调函数；
+ 穿插在启动和结束中间的物件(fn)
  + 这个中间件接受三个参数：
    + req
    + res
    + next函数（express中定义的用来执行下一个中间件的函数）

优点：

+ 模块化
  + 每个功能都能通过一个函数来实现；
  + 然后可以利用`app.use`将函数整合起来；
  + 可以单独放到一个文件，再通过`module.export`导出，实现了模块化

还有好多好多API，看文档就好

## `koa`框架

### koa的中间件模型

社区非常喜欢把`koa`的中间件模型叫做洋葱模型。

> 本质上就是函数调用栈的执行顺序

```js
app.use(async (ctx, next) => {
  
  await next(); 
  const time = ctx.response.get('X-Response-Time');//读取res.header
  console.log(`${ctx.url} - ${time}`);
});

app.use(async (ctx, next) => {
  const start = Date.now();//记录开始时间
  await next();
  const time = Date.now() - start;//记录结束时间
  ctx.set('X-Response-Time', `${time}ms`);//写入res header
});

app.use(async ctx => {
  ctx.body = 'Hello World';
  // 最后一个中间件可以不写 await next()
});
```

> 一个八卦：经常看到的洋葱模型的图，实际上是python的一个web框架Pylons的（2010）...

还有好多好多API，看文档就好

















































