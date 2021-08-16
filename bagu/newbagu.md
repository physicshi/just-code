## 手写函数Foo()

要求

```js
var a = new Foo() // => {id: 1}
var b = new Foo() // => {id: 2}
```

这个题的重点是保存状态，可以放在原型上，也可以用闭包来做：

原型版本：

```js
Foo.prototype.id=1
functin Foo(){
    this.id=Foo.prototype.id++
}
```

闭包版本：

```js
let Foo()=(function(){
    let id = 0;
    function Foo(){
        this.id=++id
    }
    return Foo
})()
```

## 组件渲染几次

```js
const FC = () => {
 const [loading, setLoading] = useState();
 useEffect(()=>{   
     setTimeout(()=>{
     setLoading(false);
     setLoading(true);
     },0)
 },[]);
 console.log(1);
 return null;
}
```

3次

### 解答

`react`内部通过 `isBatchingUpdates` 来判断`setState` 是先存进 `state` 队列还是直接更新，如果值为 `true` 则执行异步操作，为 `false` 则直接更新。

+ 在 `React` 生命周期事件和合成事件中，都会走合并操作，延迟更新的策略。看起来就像异步了

> 因为会先存进队列里，等到最后再统一执行，所以看起来像是异步的。

+ 原生事件，具体就是在 `addEventListener` 、`setTimeout`、`setInterval` 等事件中，就只能同步更新。

```js
// 每次点击时，React 只执行一次渲染，尽管你设置了两次状态
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(c => c + 1); // Does not re-render yet
    setFlag(f => !f); // Does not re-render yet
    // React will only re-render once at the end (that's batching!)
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}
```

## 用js实现一个带并发限制的异步调度器Scheduler，同时保证运行的任务最多有两个

```js
class Scheduler {
  add(promiseCreator){...}
  // ...
}

const timeout = (time) => new Promise(resolve => {
  setTimeout(resolve, time)
})
const scheduler = new Scheduler()
const addTask = (time, order) => {
  scheduler.add(() => timeout(time)).then(()=>console.log(order))
}

addTask(1000, '1')
addTask(500, '2')
addTask(300, '3')
addTask(400, '4')

// output: 2 3 1 4
// 一开始，1、2两个任务进入队列
// 500ms 时，2完成，输出2，任务3进队
// 800ms 时，3完成，输出3，任务4进队
// 1000ms 时，1完成，输出1
```

背景：js是单线程，并不存在真正的并发，但是由于宿主环境的Event Loop机制，使得异步函数调用有了“并发”这样的假象。

类似于切片执行（因为异步，多个事件回调可以在同一时间段执行）

但是有的时候我们需要控制最大的并发数量，不然在瞬间发出几十万`http`请求（`tcp`连接数不足可能造成等待），或者堆积了无数调用栈导致内存溢出。

### 解答

思路：`用一个队列缓存请求，在执行前计数+1，发送请求，当计数达到2，也就是达到并发上限，将后面的请求存入队列，每次任务执行完进入then的逻辑后计数-1，这意味着可以继续请求，即取出队列头部元素并执行`

```js
class Scheduler {
    constructor() {
        this.queue = []
        this.count = 0
    }
    add(promiseCreator) {
        let _resolve
        const tempFunc = () => {
            this.count++
            promiseCreator()
              .then(() => {
                _resolve()
                this.count--
                if (this.queue.length && this.count <= 2) {
                    this.queue.shift()()
                }
            })
        }
        if (this.count < 2 && this.queue.length === 0) {
            tempFunc()
        } else {
            this.queue.push(tempFunc)
        }

        return new Promise((resolve) => {
            _resolve = resolve
        })
    }
}

const timeout = (time) => new Promise(resolve => {
    setTimeout(resolve, time)
})

const scheduler = new Scheduler()
const addTask = (time, order) => {
    scheduler.add(() => timeout(time)).then(() => console.log(order))
}

addTask(1000, '1')
addTask(500, '2')
addTask(300, '3')
addTask(400, '4')
```

## 实现一个lazyMan

```js
LazyMan('Jack').eat('lunch').sleep(1).eat('dinner').sleepFirst(2)

// 2s 后
// I'm Jack、eat lunch
// 1s 后
// eat dinner
```

### 解答

思路：重点是实现执行下一个的next。还要在每一个方法里实现fn的打印函数以及调用next（保证能继续调用下一个），并将fn push到执行队列里，next函数是取出头部元素并执行。

为了最开始能调用next，要在构造函数里放一个setTimeout调用next，在执行完所有同步任务把fn放进队列后，开始执行setTimeout，这样就能一直调用next，依次取出队列头部函数并执行。

```js
class LazyManClass {
  // 构造函数
  constructor(name){
    this.name = name
    // 定义一个数组存放执行队列
    this.queue = []
    console.log(`Hi I am ${name}`)
    // 在调用LazyManClass时首先会打印 Hi I am ${name}
    setTimeout(() => {
      this.next()
    }, 0)

  }
  //  定义原型方法
  eat (food) {
    var fn = () => {
      console.log(`I am eating ${food}`)
     this.next()
    }
    this.queue.push(fn)
    return this
  }
  sleep (time) {
    var fn = () => {
      // 等待了10秒...
      setTimeout(() => {
        console.log(`等待了${time}秒`)
        this.next()
      }, 1000*time)
    }
    this.queue.push(fn)
    return this
  }
  sleepFirst (time) {
    var fn = () => {
      // 等待了5秒...
      setTimeout(() => {
        console.log(`等待了${time}秒`)
        this.next()
      }, 1000*time)
    }
    this.queue.unshift(fn)
    return this
  }
  next () {
    var fn = this.queue.shift()
    fn && fn()
  }
}
function LazyMan (name) {
  return new LazyManClass(name)
}
LazyMan('Tony').eat('lunch').eat('dinner').sleepFirst(5).sleep(10).eat('junk food')
// Hi I am Tony
// 等待了5秒...
// I am eating lunch
// I am eating dinner
// 等待10秒
// I am eating junk food
```

## Cash类

```js
class Cash{
    //code here
}

// 实现的功能：
const cash1 = new Cash(105);
const cash2 = new Cash(66);

const cash3 = cash1.add(cash2);
const cash4 = Cash.add(cash1,cash2);
const cash5 = Cash.add(cash1 + cash2);

console.log(`${cash3}`,`${cash3}`,`${cash3}`);
```

输出结果如下：

1元7角1分 1元7角1分 1元7角1分

```js
class Cash {
    constructor(money) {
        this.money = money;
    }
    static add(begin = 0, ...cash) {
        return new Cash( cash.reduce((acc, val) => acc + val, begin) );
    }
    add(...cash) {
        return Cash.add(this, ...cash);
    }
    static format(money) {
        let yuan = money / 100|0,
        jiao = (money / 10) % 10|0,
        fen = money % 10;
        return `${yuan}元${jiao}角${fen}分`;
    }
    toString() {
        return Cash.format(this.money);
    }
    valueOf() {
        return this.money;
    }
}
```

备注

```js
// 每个对象都有一个 toString() 方法，
// 当对象被表示为文本值时或者当以期望字符串的方式引用对象时，
// 该方法被自动调用。对对象x，toStrig() 返回 “[object type]”,
// 其中type是对象类型。如果x不是对象，toString() 返回x应有的文本值。

var obj = {};
var a = obj.toString();
console.log(a); // [Object, Object]

var arr = [1, 2, 3];
console.log(arr.toString()); // 123


// 默认情况下, valueOf() 会被每个对象Object继承。
// 每一个内置对象都会覆盖这个方法为了返回一个合理的值，
// 如果对象没有原始值，valueOf() 就会返回对象自身
var x = {};
x.valueOf = function(){
    return 10;
}
console.log(x+1);// 11
console.log(x+"hello");// 10hello
```

## useMemo和useCallback

首先这两个hooks都是为了避免多余的性能开销，两个都可以起缓存的效果，一个缓存的是函数，一个缓存函数返回的结果；就导致这两个的hooks使用场景不同，useCallback针对的是子组件的渲染（减少子组件的渲染次数）、useMemo针对的是当前组件多余的计算（当然useMemo也可以针对父子组件传值减少渲染次数）。

因为重新渲染的时候，对于函数组件是要重新执行的，这样内部有一些函数的重新执行可能会造成子组件的重复渲染或者本身这个函数的计算量就很大。可能我们并不需要他们重新执行（数据驱动视图，多余计算并没有造成视图的改变，这样就是浪费性能）

两个hooks都需要传入callback以及依赖数组deps，不过对于useMemo来说，传入的callback要有返回值；依赖数组就是callback依赖的自由变量。

> 很多情况useCallback会造成负优化，在没有自由变量时，使用 `useCallback` 后每次执行到这里内部要要比对 `inputs` 是否变化，还有存一下之前的函数，消耗更大了。

## useEffect和useLayoutEffect

唯一的区别就是

+ `useEffect`会在渲染的内容更新到DOM上后执行，不会阻塞DOM的更新

+ `useLayoutEffect`会在渲染的内容更新到DOM上之前执行，会阻塞DOM的更新

## useState和useReducer

`useReducer`是`useState`的一种替代方案。按照官方的说法：对于复杂的`state`操作逻辑，嵌套的`state`的对象，推荐使用`useReducer`。

```js
const [state, dispatch] = useReducer(reducer, initState);
```

`useReducer`接收两个参数：

第一个参数：`reducer`函数。第二个参数：初始化的`state`。返回值为最新的state和`dispatch`函数（用来触发`reducer`函数，计算对应的`state`）。

官方的例子：

```js
// 官方 useReducer Demo
    // 第一个参数：应用的初始化
    const initialState = {count: 0};

    // 第二个参数：state的reducer处理函数
    function reducer(state, action) {
        switch (action.type) {
            case 'increment':
              return {count: state.count + 1};
            case 'decrement':
               return {count: state.count - 1};
            default:
                throw new Error();
        }
    }

    function Counter() {
        // 返回值：最新的state和dispatch函数
        const [state, dispatch] = useReducer(reducer, initialState);
        return (
            <>
                // useReducer会根据dispatch的action，返回最终的state，并触发rerender
                Count: {state.count}
                // dispatch 用来接收一个 action参数「reducer中的action」，用来触发reducer函数，更新最新的状态
                <button onClick={() => dispatch({type: 'increment'})}>+</button>
                <button onClick={() => dispatch({type: 'decrement'})}>-</button>
            </>
        );
    }
```

## 封装组件需要考虑的点

+ 受控组件。用props定义组件的状态，通过onChange方法通知外部需要更新props。**不需要自己维护`state`**。这样的话props是什么样子，组件一定渲染成什么样子，一些数据也好备份，一些现场也好重现。

+ 样式独立性。自身的样式尽量写在组件内部，不另写css或者外置样式（全局主题除外，当然全局主题也要保证不影响其他第三方组件），修改组件的样式通过props传入（className、style或者各种参数）。

重点就是`受控组件`。

> 受控组件：由`react`控制的，可变状态一般保存在组件的 `state` 中，并且只能通过 `setState()`更新。

## 实现一个简单的EventEmitter类，once, on, off, emit方法

> 发布订阅模式
> 
> - on(event,fn)：监听event事件，事件触发时调用fn函数；
> - once(event,fn)：为指定事件注册一个单次监听器，单次监听器最多只触发一次，触发后立即解除监听器；
> - emit(event,arg1,arg2,arg3...)：触发event事件，并把参数arg1,arg2,arg3....传给事件处理函数；
> - off(event,fn)：停止监听某个事件。

```js
class EventEmitter{
    constructor(){
        this.handlers = {}
    }
    on(event,cb){
        if(!this.handlers[event]){
            this.handlers[event] = []
        }
        this.handlers[event].push(cb)
    }
    emit(event,...args){
        this.handlers[event].forEach((callback)=>{
            callback(...args)
        })
    }
    off(event,cb){
        this.handlers[event].splice(this.handlers[event].indexOf(cb),1)
    }
    once(event,cb){
        const fn = (...args) => {
            cb.call(this,...args)
            this.off(event,cb)
        }
        this.on(event,fn)
    }
}
```

## let、var

var声明的变量可以在全局作用域或者函数作用域中使用，let声明的变量限制在块级作用域中；在程序顶部声明时，var会向全局对象添加属性；var会存在变量提升。let存在暂时性死区；var在同一作用域可以重复声明，而let不可以，会报错；

## JS中的继承

### 原型链继承

将子类的原型设置为父类的实例

缺点：所有子类的实例会共享父类对象上的所有引用类型的属性

> 子类实例共享同一个原型对象

```js
function Parent() {
  this.lastName = 'White'
  this.dress = {
    TShirt: 'white',
    pants: 'blue'
  }
}

Parent.prototype.getName = function () {
  console.log(this.lastName);
}

function Child() {
}
Child.prototype = new Parent()

var Child1 = new Child()
var Child2 = new Child()
Child1.lastName = 'Blue'
Child1.dress.TShirt = 'yellow'

console.log(Child2.lastName) // white
console.log(Child2.dress.TShirt) // yellow
```

### 构造函数继承——call

在子类的构造函数中调用父类的构造函数并改变父类构造函数的 this 指向，从而可以使子类实例拥有父类实例的所有属性

缺点：父类在原型上的属性，子类无法访问

```js
function Parent(firstName = 'Brown') {
  this.firstName = firstName
  this.lastName = 'White'
}
function Child(name) {
  Parent.call(this, name)
}
var Child1 = new Child('HH')
var Child2 = new Child('WW')
```

### 组合继承

在子类中调用父类构造函数并改变this指向，然后将子类的原型对象设置为父类的实例

缺点：调用了两次父类的构造函数。在子类实例创建时调用了一次父类的构造函数，修改子类原型式调用了一次父类的构造函数

```js
function Parent(firstName = 'Brown') {
  this.firstName = firstName
  this.lastName = 'White'
}
Parent.prototype.getName = function () {
  console.log(`${this.firstName}·${this.lastName}`)
}

function Child(name) {
  Parent.call(this, name)
}
Child.prototype = new Parent()
Child.prototype.constructor = Child
```

### 原型式继承——模拟`Object.create`

返回值的proto是入参

缺点：和原型链继承一样，所有子类的实例会共享父类对象上的所有引用类型的属性

```js
function CreateObj(_proto_) {
  function F() { }
  F.prototype = _proto_
  return new F()
}

function Parent(){}
const Parent1 = new Parent()
const Child1 = CreateObj(Parent1)
const Child2 = CreateObj(Parent1)
```

### 寄生式继承

和原型式继承一样，共享引用值，但是对于普通对象的继承方式来说，寄生式继承相比于原型式继承，还是在父类基础上添加了更多的方法。

```js
function createObj(_proto_) {
  var clone = Object.create(_proto_)
  // 和原型式不同的地方
  clone.sayName = function () {
    return this.name
  }
  return clone
}
```

### 寄生组合式继承

可以取到父类实例属性方法，父类原型属性方法；解决实例共享父类实例属性的问题；父类构造函数只使用一次；

```js
function Parent(firstName) {
  this.firstName = firstName;
  this.lastNaem = 'White'
}

function Child(firstName) {
  // 关键代码
  Parent.call(this, firstName)
}
// 关键代码
Child.prototype = Object.create(Parent.prototype)
// 关键代码
Child.prototype.constructor = Child
```

## 箭头函数

**原型**

- 箭头函数没有`prototype`

**this指向**

- 箭头函数本身没有`this`。
  
  > `this` 指向定义时所在外层的第一个普通函数，会随该函数 `this` 指向的改变而改变

- 普通函数的 `this` 取决于调用时的情况

**能否修改 this**

- 箭头函数不能直接修改它的this指向，可以去修改被继承的普通函数的this指向来间接修改
- 普通函数可以通过 `call`，`apply`，`bind` 直接修改 `this`

**全局作用域的 this 指向**

- 箭头函数在严格和非严格模式下都绑定到 `window`
- 普通函数在严格模式下绑定到 `undefined`，否则绑定到全局对象 `window`。

## 自定义hooks

注意的点：

每一次更新函数组件都会重新执行，要保证`timer`是可靠的。

**useRef 每次都会返回相同的引用。**

每次组件重新渲染，都会执行一遍所有的`hooks`，这样`debounce`高阶函数里面的`timer`就不能起到缓存的作用（每次重渲染都被置空）。`timer`不可靠，`debounce`的核心就被破坏了。使用`useRef`的目的就是为了解决这个问题。

### 防抖

防抖关心的时最后一次的执行

> 所以要不断清除"时间戳"

```js
function useDebounce(fn, delay, dep = []) {
  const { current } = useRef({ fn, timer: null });

  return useCallback(function f(...args) {
    if (current.timer) {
      clearTimeout(current.timer);
    }
    current.timer = setTimeout(() => {
      current.fn.call(this, ...args);
    }, delay);
  }, dep)
}
```

### 节流

节流关心的事第一次的执行

```js
function useThrottle(fn, delay, dep = []) {
  const { current } = useRef({ fn, timer: null });

  return useCallback(function f(...args) {
    if (!current.timer) {
      current.timer = setTimeout(() => {
        delete current.timer;
      }, delay);
      current.fn.call(this, ...args);
    }
  }, dep);
}
```
