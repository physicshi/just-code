[toc]





## 写在前面

**所有的内容均以`react v17.0.2`为基础。**

> **多看看前端框架源码。**悄悄读源码，然后惊艳所有人。

注意：**React17并没有包含新特性！**最显著的变化：全新的事件机制+全新的`JSX`转换机制。

关于第一点，官网的图可以很好的说明：



对于第二点，我们都很熟悉以前的JSX转换机制，也就是：

在`react v17`中，

OK，接下来可以读源码了。

## `React`中的数据结构

> 可以回过头来再看

### Fiber

> `Fiber`对应一个组件需要被处理或者已经处理了，一个组件可以有一个或者多个`Fiber`

`Fiber`结构定义在`packages/react-reconciler/src/ReactInternalTypes.js`文件中：

```js
export type Fiber = {|
  // 标记不同的组件类型
  tag: WorkTag,
  // ReactElement中的key，作为唯一标识符
  key: null | string,
  // ReactElement.type的值会作为createElement的第一个参数，在整个调和过程中保证不变
  elementType: any,
  //  异步组件resolved之后返回的内容，一般是`function`或者`class`
  type: any,
  // 实例对象, 如类组件的实例、原生 dom 实例, 而 function 组件没有实例, 因此该属性是空
  stateNode: any,
  // 指向他在Fiber节点树中的`parent`，用来在处理完这个节点之后向上返回
  return: Fiber | null,

  // 单链表树结构
  // 指向自己的第一个子节点                 
  child: Fiber | null,
  // 指向自己的兄弟节点，兄弟节点的return指向同一个父节点
  sibling: Fiber | null,
  index: number,

  ref:
    | null
    | (((handle: mixed) => void) & {_stringRef: ?string, ...})
    | RefObject,

  // 新的变动带来新的props
  pendingProps: any, 
  // 上一次渲染完成之后的props                                  
  memoizedProps: any, // The props used to create the output.

  // 该Fiber对应的组件产生的Update会存放在这个队列里面
  updateQueue: mixed,

  // 上一次渲染完成之后的state
  memoizedState: any,

  // 该fiber的依赖项（上下文或者事件）
  dependencies: Dependencies | null,

  // 当前组件及子组件处于何种渲染模式 详见 TypeOfMode
  mode: TypeOfMode,

  // 用来记录side effect
  flags: Flags,
  subtreeFlags: Flags,
  deletions: Array<Fiber> | null,
                                    
	// 单链表用来快速查找下一个side effect
  nextEffect: Fiber | null,
	
  // 子树中第一个side effect                                
  firstEffect: Fiber | null,
  // 子树中最后一个side effect                                  
  lastEffect: Fiber | null,
	
  // 和优先级相关（在新的Fiber架构中很重要）                                 
  lanes: Lanes,
  childLanes: Lanes,

  alternate: Fiber | null,

  actualDuration?: number,

  actualStartTime?: number,

  selfBaseDuration?: number,

  treeBaseDuration?: number,
	// 仅在开发环境下
  _debugID?: number,
  _debugSource?: Source | null,
  _debugOwner?: Fiber | null,
  _debugIsCurrentlyTiming?: boolean,
  _debugNeedsRemount?: boolean,

  _debugHookTypes?: Array<HookType> | null,
|};

```



## `ReactDOM.render`——一切的开始

```js
ReactDOM.render(<App/>,document.getElementById("root"))
//ReactDom.render(element,container,[,callback])
```

这是`react`的开始。其中`element`是要渲染的`react`元素，`container`是容器，也就是要挂载到的真实的`DOM`节点`<div id="root"></div>`，第三个是一个可选的回调函数，在`react`初次渲染或者更新完成后，如果有`callback`，就会执行`callback`回调函数。

### `render`函数

`render`函数是渲染入口，先走一遍渲染的流程，后面再站在更高的角度来看`react`。

`render`函数定义在` packages/raect-dom/src/client/ReactDom.js`文件下，

实际上`render`函数做的事比较简单，一个是检验`container`参数，另一个是返回`legacyRenderSubtreeIntoContainer`的调用：

```js
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function,
) {
  //判断容器是否是真实的dom节点
  invariant(
    isValidContainer(container),
    'Target container is not a DOM element.',
  );
  //开发环境
  if (__DEV__) {
    const isModernRoot =
      isContainerMarkedAsRoot(container) &&
      container._reactRootContainer === undefined;
    if (isModernRoot) {
      console.error(
        'You are calling ReactDOM.render() on a container that was previously ' +
          'passed to ReactDOM.createRoot(). This is not supported. ' +
          'Did you mean to call root.render(element)?',
      );
    }
  }
  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    false,
    callback,
  );
}
```



#### isValidContainer函数

这个函数用来检验`container`参数。

**函数位于`packages/react-dom/src/client/ReactDOMRoot.js`文件中**。

函数会返回一个布尔值，判断`node`是否是符合要求的`DOM`节点：

 * `node`可以是元素节点
 * `node`可以是 document 节点
 * `node`可以是 文档碎片节点
 * `node`可以是注释节点但注释内容必须是 `react-mount-point-unstable`

   >  `react` 内部会找到注释节点的父级 通过调用父级元素的 `insertBefore` 方法，将 `element` 插入到注释节点的前面

```js
export function isValidContainer(node: mixed): boolean {
  return !!(
    node &&
    (node.nodeType === ELEMENT_NODE ||
      node.nodeType === DOCUMENT_NODE ||
      node.nodeType === DOCUMENT_FRAGMENT_NODE ||
      (node.nodeType === COMMENT_NODE &&
        (node: any).nodeValue === ' react-mount-point-unstable '))
  );
}
```

#### `legacyRenderSubtreeIntoContainer`函数

> 实际上这个函数做的事就是初始化`FiberRoot`。

接下里进入`legacyRenderSubtreeIntoContainer`的逻辑（同样在` packages/raect-dom/src/client/ReactDom.js`文件下）：

```js
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {
  if (__DEV__) {
    topLevelUpdateWarnings(container);
    warnOnInvalidCallback(callback === undefined ? null : callback, 'render');
  }

  // TODO: Without `any` type, Flow says "Property cannot be accessed on any
  // member of intersection type." Whyyyyyy.
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // 初次挂载
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 初次挂载，非批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 更新
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

`legacyRenderSubtreeIntoContainer`函数会判断是否是初次渲染：

+ 若是初次渲染则创建`FiberRoot`节点，然后就会非批量执行`updateContainer`；
+ 如果不是初次渲染，则直接执行`updateContainer`；

> 注意这两次都会判断`callback`是否存在`if (typeof callback === 'function'){//...do something}`，存在的话，则改变`callback`的`this`指向（指向getPublicRootInstance(fiberRoot)，这个后面细说），最终传参给`updateContainer`。

以上是这是总体的逻辑。下面分析细节。

### 初次渲染

#### 初始化FiberRoot

还是在`legacyRenderSubtreeIntoContainer`的逻辑里。

初次渲染是非批量更新的，这是因为批量更新是异步的可以被打断，但是初次渲染需要尽快完成、不能被打断，所以不执行批量更新。

初次渲染`root`为`undefined`，所以`root = container._reactRootContainer=legacyCreateRootFromDOMContainer(container,forceHydrate,);`

其中`legacyCreateRootFromDOMContainer`函数在` packages/raect-dom/src/client/ReactDomLegacy.js`文件中，主要做的就是在最后返回`createLegacyRoot`的调用，而`createLegacyRoot`函数在` packages/raect-dom/src/client/ReactDomRoot.js`文件中：

```js
export function createLegacyRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMLegacyRoot(container, options);
}
```

可以看到其内部实例化了`ReactDOMLegacyRoot`（同样在` packages/raect-dom/src/client/ReactDomRoot.js`文件中）：

```js
function ReactDOMLegacyRoot(container: Container, options: void | RootOptions) {
  this._internalRoot = createRootImpl(container, LegacyRoot, options);
}
```

可以看到该实例的属性 `_internalRoot` 是 `createRootImpl(container, tag, options)`，函数`createRootImpl`同样在` packages/raect-dom/src/client/ReactDomRoot.js`文件中：

```js
function createRootImpl(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  // 省略了其他逻辑
  const root = createContainer(
    container,
    tag,
    hydrate,
    hydrationCallbacks,
    strictModeLevelOverride,
  );
	// 省略了其他逻辑
  return root;
}
```

进入`createContainer`的逻辑，函数在`packages/react-reconciler/src/Reconciler.js`中：

```js
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  strictModeLevelOverride: null | number,
): OpaqueRoot {
  return createFiberRoot(
    containerInfo,
    tag,
    hydrate,
    hydrationCallbacks,
    strictModeLevelOverride,
  );
}
```

createFiberRoot函数在`packages/react-reconciler/src/ReactFiberRoot.js`里：

```js
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  strictModeLevelOverride: null | number,
): FiberRoot {
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
//省略中间判断逻辑
  return root;
}
```

可以看到最后是实例化了 `FiberRootNode(containerInfo, tag, hydrate)` 构造函数，也就是 `FiberRoot` 。

至此，我们走完了初次渲染的第一步：初始化`FiberRoot`；

#### 非批量更新

> 实际上，下面这段逻辑我不是很清楚

```js
// packages/raect-reconciler/src/ReactFiberWorkLoop.old.js(ReactFiberWorkLoop.new.js)
export function unbatchedUpdates<A, R>(fn: (a: A) => R, a: A): R {
  const prevExecutionContext = executionContext;
  executionContext &= ~BatchedContext;
  executionContext |= LegacyUnbatchedContext;
  try {
    return fn(a);
  } finally {
    executionContext = prevExecutionContext;
    if (executionContext === NoContext) {
      // Flush the immediate callbacks that were scheduled during this batch
      resetRenderTimer();
      flushSyncCallbackQueue();
    }
  }
}
```

然后是`updateContainer`的逻辑，首先是创建`update`，`payload`的参数就是`element`（也就是挂载在根节点的组件），即要渲染的`React`元素。

**这一步主要做的就是创建update任务更新队列**

> 函数位于`packages/raect-reconciler/src/ReactFiberReconciler.old.js(ReactFiberReconciler.new.js)`

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  
  //...开发环境下的判断逻辑
  
  const update = createUpdate(eventTime, lane);
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    
    //...开发环境下的判断逻辑
    
    update.callback = callback;
  }
	//将update加入updateQueue
  enqueueUpdate(current, update, lane);
  //调度更新
  const root = scheduleUpdateOnFiber(current, lane, eventTime);
  if (root !== null) {
    entangleTransitions(root, current, lane);
  }
  return lane;
}
```

### setState

`this.setState`内会调用`this.updater.enqueueSetState`方法。

```js
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```







