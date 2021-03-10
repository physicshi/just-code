# Axios的基本使用

```js
//引入
import axios from 'axios'
```

## 请求方式

首先Axios支持多种请求方式：

+ `axios(config)`或者`axios.reauest(config)`
+  `axios[method]()`

注意：`request()`是axios中最重要的方法之一，上面所有的调起请求的方式都是基于这个方法，也是我们下面学习源码重点解读的部分。

### `axios(config)`/`axios.request(config)`

#### 使用方法

```js
axios({
  method: 'post',
  url: '/user/12345',
  data: {
    firstName: 'Fred',
    lastName: 'Flintstone'
  }
});
```

#### 源码

可以看到`axios(config)`本质上是调用了`createInstance(config)`方法生成的Axios实例，最终内部通过是在调用`Axios`的`request`方法：

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);
  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);
  // Copy context to instance
  utils.extend(instance, context);
  return instance;
}
// Create the default instance to be exported
var axios = createInstance(defaults);
// Expose Axios class to allow class inheritance
axios.Axios = Axios;
```

### `axios[method]()`

可以根据请求的`method`，可以选择对应的请求方法，比如get请求可以使用`axios.get`，post请求可以使用`axios.post`。

#### 使用方法

```js
 axios.get("https://httpbin.org/get", {
      params: {
        name: "xiaoming",
        age: 18
      }
    }).then(res => {
      console.log(res);
    }).catch(err => {
      console.log(err);
    });
```

#### 源码

可以看到，这些方法其实都是封装了`axios.request()`方法的，只是把传入的参数中`method`固定下来。

```js
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```

### `axios.create()`

这个方法不是能直接发起请求的，而是创建一个Axios实例，然后在通过实例的`request()`方法或者其他实例上的方法去发起请求。

```
// Create an instance using the config defaults provided by the library
// At this point the timeout config value is `0` as is the default for the library
const instance = axios.create();

instance.get('/longRequest', {
  timeout: 5000
});
```

## config

默认配置 + 用户配置 = 最终配置。

首先axios存在默认配置（位于源码的defaults.js文件中），主要是设置默认的请求头、格式化请求正文以及响应正文：

设置默认请求头：

![image-20210309212832101](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210309212832101.png)

根据当前环境，获取默认的请求方法：

![image-20210309212948006](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210309212948006.png)

这是默认配置，再来看用户配置，因为我们知道这些请求方式本质都是在调用`request`，`request`方法的参数其实才是我们说的用户配置。接下来进入`request`的源码。

`request`部分主要完成合并配置，以及将拦截器的函数入栈`chain`，然后用`Promise`来执行。

首先看合成配置，之前提到过请求的配置除了传参进来的`config`之外，还有在`axios.defaults`中的配置。所以，需要先拿到`config`，然后跟`axios.defaults`得到本次请求的配置。

> lib/core/Axios.js

我们知道**`arguments`** 是传递给函数的参数组成的类数组对象。这里的逻辑是先判断config是否是string类型，如果是，第二个传参才是config（第一个传参是url），否则，第一个传参就是config

> ```js
> // 第一种传参方式
> axios.request('/api', {
>   method: 'get'
>   // ...
> })
> // 第二种传参方式
> axios.request({
>   url: '/api',
>   method: 'get'
>   // ...
> })
> ```

![image-20210310100045715](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210310100045715.png)

这是用户配置的部分，接下来是合并配置：默认配置+用户配置=最终配置；

```js
config = mergeConfig(this.defaults, config);
```

该方法在`lib/coremergeConfig.js`中，主要完成了深拷贝以及默认值处理。









































