一直以来写typescript都比较随意，现在规范一下。

来看一下React源码中是的useState的声明：

```ts
// ...
/**
 * Returns a stateful value, and a function to update it.
 *
 * @version 16.8.0
 * @see https://reactjs.org/docs/hooks-reference.html#usestate
 */
function useState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>];
// convenience overload when first argument is ommitted
/**
 * Returns a stateful value, and a function to update it.
 *
 * @version 16.8.0
 * @see https://reactjs.org/docs/hooks-reference.html#usestate
 */
function useState<S = undefined>(): [S | undefined, Dispatch<SetStateAction<S | undefined>>];
// ...
```

在React源码中，useState有两个函数。第二个函数重写（override）了第一个函数，使我们可以在调用useState时可以不用直接声明state变量类型。

值得注意的是，第一个方法接一个名为S的TypeScript泛型，通过它，我们可以定义state的变量类型。

## useState的基础用法

以下是在TypeScript中使用useState的基本例子：

```ts
import React, {useState} from 'react'

const App = () => {
  const [name, setName] = useState<string>('未定义变量')
  const [age, setAge] = useState<number>(28)
  const [isProgrammer, setIsProgrammer] = useState<boolean>(true)

  return (
    <div>
      <ul>
        <li>Name: {name}</li>
        <li>Age: {age}</li>
        <li>Programmer: {isProgrammer ? 'Yes' : 'No'}</li>
      </ul>
    </div>
  )
}
```

不过对于基础类型，可以不用声明变量，typescript会判断相应的变量类型。

## useState进阶用法

当我们需要使用useState处理复杂数据时（如数组或者对象），我们需要借用TypeScript中接口（interface）这个概念。

假设我们需要在useState中声明如下数据：

```js
[
  {
    "id": 1,
    "name": "蔡文姬",
    "type": "辅助"
  },
  {
    "id": 1,
    "name": "后裔",
    "type": "射手"
  },
  {
    "id": 1,
    "name": "鲁班7号",
    "type": "射手"
  }
]
```

我们需要首先定义一个代表这组数据类型的接口（interface）。

```ts
interface IChampion {
  id: number;
  name: string;
  type: string;
}
```

然后我们在使用useState中，声明我们的state为Champion类型变量。

```tsx
import React, {useState} from 'react'

interface IChampion {
  id: number;
  name: string;
  type: string;
}

export default function Champions() {
  const [champions, setChampions] = useState<IChampion[]>([
    {
      id: 1,
      name: '蔡文姬',
      type: '辅助'
    },
    {
      id: 1,
      name: '后裔',
      type: '射手'
    },
    {
      id: 1,
      name: '鲁班7号',
      type: '射手'
    }
  ])

  return (
    <div>
      <ul>
        {champions.map(champion=> (
          <li key={champion.id}>{champion.name} - {champion.type}</li>
        ))}
      </ul>
    </div>
  )
}
```

但通常，我们会从API中异步获取数据，而非在useState中直接声明变量。

```tsx
import React, {useState, useEffect} from 'react'
import axios from 'axios'

interface IChampion {
  id: number;
  name: string;
  type: string;
}

export default function Champions() {
  const [champions, setChampions] = useState<IChampion[]>([])

  useEffect(() => {
    axios.get<IUser[]>('https://api.yourservice.com/champions')
      .then(({ data }) => {
        setChampions(data)
      })
  }, [])

  return (
    <div>
      <ul>
        {champions.map((champion: IChampion) => (
          <li key={champion.id}>{champion.name} - {champion.type}</li>
        ))}
      </ul>
    </div>
  )
}
```

以上就是在React TypeScript，useState的一些用法。

## 附：Promise类型定义

之所以介绍这个是因为我们在写代码时总会遇到各种请求相关的类型定义，不过我们都可以**去阅读源码中类型的声明来获知类型定义的信息**：

```ts
/**
     * Creates a new Promise.
     * @param executor A callback used to initialize the promise. This callback is passed two arguments:
     * a resolve callback used resolve the promise with a value or the result of another promise,
     * and a reject callback used to reject the promise with a provided reason or error.
     */
    new <T>(executor: (resolve: (value?: T | PromiseLike<T>) => void, reject: (reason?: any) => void) => void): Promise<T>;
```

Promise的类型定义如上，我们可以看到 Promise 返回值的类型定义，可以由两部分决定。第一个是构造时的泛型值，第二个是 reslove函数 value值得类型。

一般我们会利用**构造时的范型值声明返回值的范型类型**：

```ts
// model.d.ts
export interface IMessageboolean {
  data: boolean;
  host: string;
  message: string;
  port: number;
  result: number;
  status: number;
  timestamp: string;
}

// api.ts
export async function recordUsingPOST(params: { tipName?: string }) {
  const result = await axios.post<model.IMessageboolean>(`/jiekou/jiekou`, params);
  return result.data;
}
```

