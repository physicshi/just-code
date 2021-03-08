## Axios的基本使用

```js
//引入
import axios from 'axios'
```

首先Axios支持多种请求方式：

+ `axios(config)`或者`axios.reauest(config)`
+  `axios[method]()`

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


