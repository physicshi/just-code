## 初次渲染——一切的开始

```js
ReactDOM.render(<App/>,document.getElementById("root"))
```

这是react的开始，主要的任务是

+ 创建`ReactRoot`对象，这是站在`react`顶点的对象，

+ 创建`FiberRoot`和`RootFiber`

+ 创建更新

```js
// packages/raect-dom/src/client/ReactDom.js

```

