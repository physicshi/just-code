- [设计表单](#设计表单)
- [表单组件层](#表单组件层)
  - [Form](#form)
  - [FormItem](#formitem)
  - [表单控件](#表单控件)
- [状态管理层](#状态管理层)
  - [表单的状态层？](#表单的状态层)
  - [表单状态层保存在哪里？](#表单状态层保存在哪里)
- [数据通信层](#数据通信层)
  - [改变状态](#改变状态)
  - [触发校验](#触发校验)
- [FormStore 表单状态管理](#formstore-表单状态管理)
- [useForm 表单状态管理 hooks 设计](#useform-表单状态管理-hooks-设计)

## 设计表单

其实表单设计的难点在于对于表单数据的管理，以及把状态分给每一个表单项。

整个表单体系的流程可以分为，**状态收集** ，**状态管理** ，**状态验证** ， **状态下发** 。

以`antd`为例：

```js
<Form  onFinish={onFinish} >
   <FormItem name="name"  label="xxx" >
       <Input />
   </FormItem>
    <FormItem name="author"  label="yyy" >
       <Input />
   </FormItem>
   <Button htmlType="submit" >确定</Button>
</Form>
```

## 表单组件层

一个完整的表单体系在组件层面应该分为三个部分： Form ，FormItem ，表单控件。

### Form

这是整个表单的最顶层，要支持状态的保存与下发。

+ **状态保存**
  
  `Form` 的作用，管理整个表单的状态，这个状态包括具体表单控件的 value，以及获取表单，提交表单，重置表单，验证表单等方法。

+ **状态下发**
  
  `Form` 不仅仅要管理状态，而且还要下发传递这些状态。把这些状态下发给每一个 FormItem ，由于考虑到 Form 和 FormItem 有可能深层次的嵌套，所以选择通过 React context 保存下发状态最佳。

### FormItem

作为Form的基本单元，收集自己表单控件的状态，传递给顶层Form进行统一管理。

- **状态收集**
  
   首先很重要的一点，就是收集表单的状态，传递给 Form 组件，比如属性名，属性值，校验规则等。

- **控制表单组件**
  
  还有一个功能就是，将 `FormItem` 包裹的组件，变成受控的，一方面能够自由传递值 value 给表单控件，另一方面，能够劫持表单控件的 change 事件，得到最新的 value ，上传状态。

- **提供Label和验证结果的 UI 层** 
  
  `FormItem` 还有一个作用就是要提供表单单元项的标签 label ，如果校验不通过的情况下，需要展示错误信息 UI 样式。

### 表单控件

表单控件设计（比如 Input ，Select 等）：

- 首先表单控件一定是与上述整个表单验证系统**零耦合**的，也就是说 Input 等控件脱离整个表单验证系统，可以独立使用。
- 在表单验证系统中，表单控件，不需要自己绑定事件，统一托管于 `FormItem` 处理。

## 状态管理层

### 表单的状态层？

**保存信息：** 首先最直接的是，需要保存表单的属性名 `name` ，和当前的属性值 `value` ，除此之外还要保存当前表单的验证规则 `rule` ，验证的提示文案 `message` ，以及验证状态 `status`。  `Promise` 的启发，引用了三种状态：

- resolve -> 成功状态，当表单验证成功之后，就会给 resolve 成功状态标签。
- reject -> 失败状态，表单验证失败，就会给 reject 失败状态标签。
- pendding -> 待验证状态，初始化，或者重新赋予表单新的值，就会给 pendding 待验证标签。

**数据结构：**

上面介绍了表单状态层保存的信息。接下来用什么数据结构保留这些信息。

```js
/*  
    TODO: 数据结构
    model = {
       [name] ->  validate  = {
           value     -> 表单值    (可以重新设定)
           rule      -> 验证规则  ( 可以重新设定)
           required  -> 是否必添 -> 在含有 rule 的情况下默认为 true
           message   -> 提示消息
           status    -> 验证状态  resolve -> 成功状态 ｜reject -> 失败状态 ｜ pendding -> 待验证状态 |
       }
   }
*/
```

- `model` 为整个 Form 表单的数据层结构。
- `name` 为键，对应 FormItem 的每一个 name 属性，
- `validate` 为 name 属性对应的值，保存当前的表单信息，包括上面说到那几个重要信息。

打个比方，最后存在 form 的数据结构如下所示：

```js
model = {
    name :{ /* xxx formItem */
        value: ...
        rule:...
        required:...
        message:...
        status:...
    },
    author:{ /* yyy formItem */
        value: ...
        rule:...
        required:...
        message:...
        status:...
    }
}
```

### 表单状态层保存在哪里？

上面说到了整个表单的状态层，那么状态层保存在哪里呢 ？

状态层最佳选择就是保存在 Form 内部，可以通过 `useForm` 一个自定义 hooks 来维护和管理表单状态实例 **`FormStore`** 。

## 数据通信层

整个表单数据通信的设计，只需要考虑两个方面：改变状态、触发验证。

### 改变状态

当系统中一个控件比如 `Input` 值改变的时候，①可以是触发了 `onChange` 方法，首先由于 `FormItem` 控制表单控件，所以 FormItem 会最先感知到最新的 value ，②并通知给 Form 中的表单管理 `FormStore` ， ③ `FormStore` 会更新状态，④ 然后把最新状态下发到对应的 `FormItem` ，⑤`FormItem` 接收到任务，再让 Input 更新最新的值，视觉感受 Input 框会发生变化 ，完成受控组件状态改变流程。

### 触发校验

表单校验有两种情况：

**第一种：** 可能是给 FormItem 绑定的校验事件触发，比如 onBlur 事件触发 ，而引起的对单一表单的校验。流程和上述改变状态相同类似。

**第二种：** 有可能是提交事件触发，或者手动触发校验事件，引起的整个表单的校验。流程首先触发 submit 事件，①然后通知给 Form 中 `FormStore`，② `FormStore` 会对整个表单进行校验，③然后把每个表单的状态，异步并批量下发到每一个 FormItem ，④ FormItem 就可以展示验证结果。

## FormStore 表单状态管理

`FormStore` 是整个表单验证系统最核心的功能了，里面包括保存的表单状态 model， 以及管理这些状态的方法，这些方法有的是对外暴露的，开发者可以通过调用这些对外的 api 实现**提交表单**，**校验表单** ， **重置表单** 等功能。

 FormStore 保存的重要属性。

- `model` ： 首先 model 为整个表单状态层的核心，绑定单元项的内容都存在 model 中，上述已经介绍了。

- `control` ：control 存放了每一个 FormItem 的更新函数，因为表单状态改变，Form 需要把状态下发到每一个需要更新的 FormItem 上。

- `callback`： callback 存放表单状态改变的监听函数。

- `penddingValidateQueue`：由于表单验证状态的下发是采用异步的，显示验证状态的更新，
  
  > 如果状态改变，把当前更新任务放在 `penddingValidateQueue` 待验证队列中。**为什么采用异步校验更新呢？** 首先验证状态改变，带来的视图更新，不是那么重要，可以先执行更高优先级的任务，还有一点就是整个验证功能，有可能在异步情况下，表单会有多个表单单元项，如果直接执行更新任务，可能会让表单更新多次，所以放入`penddingValidateQueue` 在配合 `unstable_batchedUpdates`批量更新，统一更新这些状态。

## useForm 表单状态管理 hooks 设计

上面的 `FormStore` 就是通过自定义 hooks —— `useForm` 创建出来的。useForm 可以独立使用，创建一个 `formInstance` ，然后作为 form 属性赋值给 Form 表单。 如果没有传递 默认会在 Form 里通过 `useForm` 自动创建一个。（参考 antd，用法一致 ）。

**代码实现：**

```js
function useForm(form,defaultFormValue = {}){
   const formRef = React.useRef(null)
   const [, forceUpdate] = React.useState({})
   if(!formRef.current){
      if(form){
          formRef.current = form  /* 如果已经有 form，那么复用当前 form  */
      }else { /* 没有 form 创建一个 form */
        const formStoreCurrent = new FormStore(forceUpdate,defaultFormValue)
        /* 获取实例方法 */
        formRef.current = formStoreCurrent.getForm()
      }
   }
   return formRef.current
}
```

useForm 的逻辑实际很简单：

- 通过一个 useRef 来保存 FormStore 的重要 api。
- 首先会判断有没有 form ，如果没有，会实例化 FormStore ，上面讲的 FormStore 终于用到了，然后会调用 **`getForm`** ，把重要的 api 暴露出去。
- 什么情况下有 form ，当开发者用 useForm 单独创建一个 FormStore 再赋值给 Form 组件的 form 属性，这个时候就会存在 form 了。