- [说一下react-router](#说一下react-router)
  - [history模式](#history模式)
  - [hash模式](#hash模式)

## 说一下react-router

`react-router`是基于`history`库来实现的。**本质上是维护了一个路由堆栈，底层就是对于浏览器原生api的封装。**

> `history`库提供了简洁的`API`，让你可以管理`history`堆栈、跳转、跳转前确认，以及保持会话之间的状态。

每个`router`组件创建了一个`history`对象，用来记录当前路径( `history.location` )，上一步路径也存储在堆栈中。当前路径改变时，视图会重新渲染，给你一种跳转的感觉。`history`对象有 `history.push()`和 `history.replace()`这些方法来实现当前路径的改变。当你点击 `<Link>`组件会触发 `history.push()`，使用 `<Redirect>`则会调用 `history.replace()`。其他方法 - 例如 `history.goBack()`和 `history.goForward()` - 用来根据页面的后退和前进来跳转`history`堆栈。

### history模式

`History`模式，依赖`window.history`这个api，可以获取浏览器历史记录的`history`对象。比如`window.history.back()`、`window.history.forward()`和`window.history.go()`方法，可以实现在用户历史记录中向后和向前的跳转。

**使用`history.pushState()`和`history.replaceState()`方法，它可以操作浏览器的历史栈，同时不会引起页面的刷新（可避免页面重新加载）。**

当页面在历史记录中切换，会产生`popstate`事件，可以通过`window.popstate`监听页面路由切换的情况。

### hash模式

`Hash` 模式依赖 `location` 对象的 `hash` 属性（`location.hash`）和`hashchange`事件，**location.hash的设置和获取，并不会造成页面重新加载**，利用这一点，我们可以记录页面关键信息的同时，提升页面体验。

当页面的 `hash` 改变时，`hashchange`事件会被触发，同时提供了两个属性：`newURL`（当前页面新的 `URL`）和`oldURL`（当前页面旧的 `URL`）。
