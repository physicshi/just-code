

设计`Hooks`主要是解决`ClassComponent`的几个问题：

1. 很难复用逻辑（只能用HOC，或者render props），会导致组件树层级很深
2. 会产生巨大的组件（指很多代码必须写在类里面）
3. 类组件很难理解，比如方法需要`bind`，`this`指向不明确

## 以useState为例

在React17后，每个节点都会对应一个FiberNode对象，这个对象里装载了很多属性，最重要的就是`this.memoizedState`属性：Fiber中的`memorizedStated`用来存储state。在类组件中`state`是一整个对象，可以和`memoizedState`一一对应。但是在`Hooks`中，React并不知道我们调用了几次`useState`（`useState`是一个方法，它本身是无法存储状态的），**所以React通过将一个Hook对象挂载在`memorizedStated`上来保存函数组件的`state`**：

```js
// ReactFiberHooks.js 
//Hook对象的结构
export type Hook = {
  memoizedState: any, 

  baseState: any,    
  baseUpdate: Update<any, any> | null,  
  queue: UpdateQueue<any, any> | null,  

  next: Hook | null, 
};
```

重点关注`memoizedState`和`next`

- `memoizedState`是用来记录当前`useState`应该返回的结果的
- `queue`：缓存队列，存储多次更新行为
- `next`：指向下一次`useState`对应的Hook对象。

<font color=red size=3>hook 相关的所有信息收敛在一个 hook 对象里，而 hook 对象之间以单向链表的形式相互串联。</font>

**Hooks 的本质其实是链表。**

也就是说：

```
hook1 => Fiber.memoizedState
state1 === hook1.memoizedState
hook1.next => hook2
state2 === hook2.memoizedState
hook2.next => hook3
state3 === hook2.memoizedState
```

每个在`FunctionalComponent`中调用的`useState`都会有一个对应的`Hook`对象，他们按照执行的顺序以类似链表的数据格式存放在`Fiber.memoizedState`上

**重点来了：就是因为是以这种方式进行`state`的存储，所以`useState`（包括其他的Hooks）都必须在`FunctionalComponent`的根作用域中声明，也就是不能在`if`或者循环中声明**

**最主要的原因就是你不能确保这些条件语句每次执行的次数是一样的**，也就是说如果第一次我们创建了`state1 => hook1, state2 => hook2, state3 => hook3`这样的对应关系之后，下一次执行因为`something`条件没达成，导致`useState(1)`没有执行，那么运行`useState(2)`的时候，拿到的`hook`对象是`state1`的，那么整个逻辑就乱套了，**所以这个条件是必须要遵守的！**



>  关于`this.memoizedState`：**这个`key`用来存储在上次渲染过程中最终获得的节点的`state`，每次`render`之前，React会计算出当前组件最新的`state`然后赋值给组件，再执行`render`。**注意：类组件和使用useState的函数组件均适用。



### 初次渲染

进入useState源码可以发现：

```js
export function useState<S>(initialState: (() => S) | S) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

主要的逻辑只有两行。所以重点在于`dispatcher`

![image-20210307180650056](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210307180650056.png

![image-20210307180729989](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210307180729989.png)

mounState 的主要工作是初始化 Hooks，给出Hooks对象的结构。**hook 相关的所有信息收敛在一个 hook 对象里，而 hook 对象之间以单向链表的形式相互串联。**



### 更新阶段

![image-20210307182247488](C:\Users\sgy\AppData\Roaming\Typora\typora-user-images\image-20210307182247488.png)

首次渲染和更新渲染的区别，在于调用的是 mountState，还是 updateState。mountState 初始化了hooks对象，而 hooks 对象之间以单向链表的形式相互串联； updateState 之后的是操作链路，按顺序去遍历之前构建好的链表，取出对应的数据信息进行渲染。

mountState（首次渲染）构建链表并渲染；updateState 依次遍历链表并渲染。

hooks 的渲染是通过“依次遍历”来定位每个 hooks 内容的。如果前后两次读到的链表在顺序上出现差异，那么渲染的结果自然是不可控的。所以不可以把Hooks放到if或者循环里，因为一旦某次没满足循环条件，`useState`没执行，然后进行了下一次循环，链路就被打乱了。

### 细节整理

两个过程的核心是`renderWithHooks`函数。

（源码略复杂，用nextCurrentHook的值来区分mount和update，设置不同的dispatcher，然后初始化hooks或者获取该Hook对象中的 queue进行更新。总得来说就是更新memoizedState和updateQueue）。

```js
 // 用nextCurrentHook的值来区分mount和update，设置不同的dispatcher
  ReactCurrentDispatcher.current =
      nextCurrentHook === null
      // 初始化时ReactCurrentDispatcher.current 会指向HooksDispatcherOnMount 对象：初始化hooks
        ? HooksDispatcherOnMount
        // 更新时 ReactCurrentDispatcher.current 会指向HooksDispatcherOnUpdate对象：获取该Hook对象中的 queue，遍历链表进行更新
        : HooksDispatcherOnUpdate;

```



## 总结

useState的源码非常简单，它本身作为一个函数是没办法保存状态的，我们真正的状态存储在节点上，即`FiberNode`的`memoizedState`属性上，对于类组件state可以直接和`memoizedState`对应；对于函数式组件，在useState初次调用渲染时，mount过程通过将一个Hook对象挂载在`memorizedStated`上来保存函数组件的`state`。Hook对象有一个next属性，指向下一次`useState`对应的Hook对象。**所以Hooks本质上就是链表。**更新阶段就是依次遍历链表去执行渲染。

- 当我们第一次调用`[count, setCount] = useState(0)`时，创建一个queue
- 每一次调用`setCount(x)`，就dispach一个内容为x的action（action的表现为：将count设为x)，action存储在queue中，以前面讲述的有环链表规则来维护
- 这些action最终在`updateReducer`中被调用，更新到`memorizedState`上，使我们能够获取到最新的state值。



