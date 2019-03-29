# Beginner Tutorial（已校验）

## Objectives of this tutorial

## 本手册的目的

This tutorial attempts to introduce redux-saga in a (hopefully) accessible way.

本教程的目的是希望以一种易于理解的方式向你介绍 redux-saga。

For our getting started tutorial, we are going to use the trivial Counter demo from the Redux repo. The application is quite basic but is a good fit to illustrate the basic concepts of redux-saga without being lost in excessive details.

在入门教程中，我们会采用 Redux repo 中的计数器示例来作为演示。这个示例应用非常基础，但却很好的展示了 redux-saga 中的一些基础概念，且不会让你面对太多的技术细节而感到迷茫。

### The initial setup

### 初始化

Before we start, clone the [tutorial repository](https://github.com/redux-saga/redux-saga-beginner-tutorial).

在开始之前，我们先从 [tutorial repository](https://github.com/redux-saga/redux-saga-beginner-tutorial) 拉一份代码。

> The final code of this tutorial is located in the `sagas` branch.
>
> 本教程的最终代码在 `saga` 分支上。

Then in the command line, run:

然后在命令行中执行：

```sh
$ cd redux-saga-beginner-tutorial
$ npm install
```

To start the application, run:

运行下面的命令来启动应用：

```sh
$ npm start
```

We are starting with the most basic use case: 2 buttons to `Increment` and `Decrement` a counter. Later, we will introduce asynchronous calls.

我们从最基础的使用场景来作为切入点：2 个按钮，一个 `Increment` 用来增加计数，`Decrement` 用来减少计数。然后，我们将会介绍如何进行异步调用。

If things go well, you should see 2 buttons `Increment` and `Decrement` along with a message below showing `Counter: 0`.

如果一切进展顺利，你应该会看到 2 个按钮，即 `Increment` 和 `Decrement`，在按钮下面还应有一个文字提示 `Counter: 0`。

> In case you encountered an issue with running the application. Feel free to create an issue on the [tutorial repo](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues).
>
> 如果你在运行应用的时候遇到了任何问题。请在 [tutorial repo](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues) 创建一个 issue。

## Hello Sagas!

We are going to create our first Saga. Following the tradition, we will write our 'Hello, world' version for Sagas.

我们马上就要开始创建第一个 Saga 了。遵照惯例（一般以 “Hello，world” 为第一个示例），我们将编写一个 Saga 版本的 “Hello，world”。

Create a file `sagas.js` then add the following snippet:

创建一个 `sagas.js` 文件，然后添加下面的代码：

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!')
}
```

So nothing scary, just a normal function (except for the `*`). All it does is print a greeting message into the console.

没啥好怕的，这就是一个普通函数（除了`*`）。它做的所有操作就仅仅是在控制台打印一条问候语。

In order to run our Saga, we need to:

为了让我们的 Saga 能够运行，我们需要这么做：

- create a Saga middleware with a list of Sagas to run (so far we have only one `helloSaga`)
- 将待运行的 Saga 放进一个列表中来创建 Saga 中间件（目前我们只有一个 `helloSaga`）。
- connect the Saga middleware to the Redux store
- 将 Saga 中间件与 Redux store 进行关联

We will make the changes to `main.js`:

我们将对 `main.js` 做以下修改：

```javascript
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

// ...
import { helloSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

const action = type => store.dispatch({type})

// rest unchanged
```

First we import our Saga from the `./sagas` module. Then we create a middleware using the factory function `createSagaMiddleware` exported by the `redux-saga` library.

首先我们从 `./sagas` 模块引入（import）Saga。然后使用 `redux-saga` 提供的 `createSagaMiddleware` 来创建一个 Saga 中间件。

Before running our `helloSaga`, we must connect our middleware to the Store using `applyMiddleware`. Then we can use the `sagaMiddleware.run(helloSaga)` to start our Saga.

在运行 `helloSaga` 之前，我们必须先用 `applyMiddleware` 将中间件与 Store 进行关联。然后可以使用 `sagaMiddleware.run(helloSaga)` 将我们的 Saga 启动。

So far, our Saga does nothing special. It just logs a message then exits.

目前为止，我们的 Saga 没有做任何骚操作（即特殊操作，奇淫技巧）。它仅仅是在输出一条消息之后就退出了。

## Making Asynchronous calls

## 执行异步调用

Now let's add something closer to the original Counter demo. To illustrate asynchronous calls, we will add another button to increment the counter 1 second after the click.

现在我们来添加一些与最初的计数器示例相关的代码。为了说明异步调用，我们需要再增加另一个按钮用于在它被点击完成 1 秒之后增加计数。

First things first, we'll provide an additional button and a callback `onIncrementAsync` to the UI component.

第一步，我们在 UI 组件中添加额外的按钮和按钮被点击之后需要执行的（异步）回调函数 `onIncrementAsync`。

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    <button onClick={onIncrementAsync}>
      Increment after 1 second
    </button>
    {' '}
    <button onClick={onIncrement}>
      Increment
    </button>
    {' '}
    <button onClick={onDecrement}>
      Decrement
    </button>
    <hr />
    <div>
      Clicked: {value} times
    </div>
  </div>
```

Next we should connect the `onIncrementAsync` of the Component to a Store action.

接下来我们需要把组件中的 `onIncrementAsync` 和 Store 的 action 联系起来。

We will modify the `main.js` module as follows

我们需要对 `main.js` 模块做出以下修改

```javascript
function render() {
  ReactDOM.render(
    <Counter
      value={store.getState()}
      onIncrement={() => action('INCREMENT')}
      onDecrement={() => action('DECREMENT')}
      onIncrementAsync={() => action('INCREMENT_ASYNC')} />,
    document.getElementById('root')
  )
}
```

Note that unlike in redux-thunk, our component dispatches a plain object action.

注意这里和 redux-thunk 不同的是，我们的组件 dispatch 了一个纯对象作为 action。

Now we will introduce another Saga to perform the asynchronous call. Our use case is as follows:

现在我们将引入另一个 Saga 来执行异步调用。我们的使用场景如下：

> On each `INCREMENT_ASYNC` action, we want to start a task that will do the following
>
> 在收到每个 `INCREMENT_ASYNC` action 时，我们希望启动一个任务来完成下面的事情
>
> - Wait 1 second then increment the counter
>
> - 等待 1 秒后增加计数

Add the following code to the `sagas.js` module:

给 `sagas.js` 模块增加如下代码：

```javascript
import { put, takeEvery } from 'redux-saga/effects'

const delay = (ms) => new Promise(res => setTimeout(res, ms))

// ...

// Our worker Saga: will perform the async increment task
// 工作 Saga（与工作线程类似）：将用于异步增加计数
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: spawn a new incrementAsync task on each INCREMENT_ASYNC
// 监视 Saga：在每次监听到 INCREMENT_ASYNC action 的时候产生一个新的 incrementAsync 任务
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

Time for some explanations.

现在花点时间来解释一下。

We create a `delay` function that returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will resolve after a specified number of milliseconds. We'll use this function to *block* the Generator.

我们创建了一个 `delay` 函数，它返回一个将在指定毫秒之后 resolve 的 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)。我们将使用这个函数来 *阻塞* Generator。

Sagas are implemented as [Generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) that *yield* objects to the redux-saga middleware. The yielded objects are a kind of instruction to be interpreted by the middleware. When a Promise is yielded to the middleware, the middleware will suspend the Saga until the Promise completes. In the above example, the `incrementAsync` Saga is suspended until the Promise returned by `delay` resolves, which will happen after 1 second.

Saga 使用向 redux-saga 中间件 *yield* 对象的 [Generator 函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) 来作为实现。这些 yield 的对象是能够被中间件理解的一些指令。当 yield 一个 Promise 给中间件时，中间件会暂停 Saga 直到 Promise 执行完成。在上面的例子中， `incrementAsync` Saga 被暂停直到从 `delay` 返回的 Promise 被 resolve，这个操作会花上 1 秒钟。

Once the Promise is resolved, the middleware will resume the Saga, executing code until the next yield. In this example, the next statement is another yielded object: the result of calling `put({type: 'INCREMENT'})`, which instructs the middleware to dispatch an `INCREMENT` action.

一旦 Promise 被 resolve，中间件就会恢复 Saga 并继续执行到下一个 yield 的位置。在这个例子中，下一条语句又是另一个 yield 的对象：即 `put({type: 'INCREMENT'})` 的返回结果，它用于指示中间件需要 dispatch 一个 `INCREMENT` action。

`put` is one example of what we call an *Effect*. Effects are plain JavaScript objects which contain instructions to be fulfilled by the middleware. When a middleware retrieves an Effect yielded by a Saga, the Saga is paused until the Effect is fulfilled.

`put` 就是一个被我们称作 *Effect* 的示例。Effects 是纯 JavaScript 对象，其中包含了中间件需要完成的指令。当中间件检测到一个从 Saga 中 yield 出来的 Effect 时，Saga 就会被暂停直到 Effect 完成后才会继续执行。

So to summarize, the `incrementAsync` Saga sleeps for 1 second via the call to `delay(1000)`, then dispatches an `INCREMENT` action.

总而言之， `incrementAsync` Saga 通过调用 `delay(1000)` 来休眠 1 秒，然后 dispatch 一个 `INCREMENT` action。

Next, we created another Saga `watchIncrementAsync`. We use `takeEvery`, a helper function provided by `redux-saga`, to listen for dispatched `INCREMENT_ASYNC` actions and run `incrementAsync` each time.

接下来，我们创建另一个 Saga `watchIncrementAsync`。我们使用 `takeEvery`，它是 `redux-saga` 提供的一个用于监听 `INCREMENT_ASYNC` action 并在每次收到后执行 `incrementAsync` 的函数。

Now we have 2 Sagas, and we need to start them both at once. To do that, we'll add a `rootSaga` that is responsible for starting our other Sagas. In the same file `sagas.js`, refactor the file as follows:

现在有两个 Saga ，我们需要同时启动它们。为了达到这个目的，我们需要添加一个 `rootSaga` 来负责启动其他 Saga。像下面这样重构 `sagas.js`：

```javascript
import { put, takeEvery, all } from 'redux-saga/effects'

const delay = (ms) => new Promise(res => setTimeout(res, ms))

function* helloSaga() {
  console.log('Hello Sagas!')
}

export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}

// notice how we now only export the rootSaga
// single entry point to start all Sagas at once
export default function* rootSaga() {
  yield all([
    helloSaga(),
    watchIncrementAsync()
  ])
}
```

This Saga yields an array with the results of calling our two sagas, `helloSaga` and `watchIncrementAsync`. This means the two resulting Generators will be started in parallel. Now we only have to invoke `sagaMiddleware.run` on the root Saga in `main.js`.

该 Saga yield 了一个包含了另外两个 saga 执行结果的数组，即 `helloSaga` 和 `watchIncrementAsync`。这意味着这两个 Generator 将会并行启动。现在我们只需要在 `main.js` 中用 root Saga 作为参数来调用 `sagaMiddleware.run`。

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## Making our code testable

## 让代码变得可测试

We want to test our `incrementAsync` Saga to make sure it performs the desired task.

我们希望能对 `incrementAsync` Saga 进行测试以确保它能按照预期执行。

Create another file `sagas.spec.js`:

创建另一个文件 `sagas.spec.js`：

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

`incrementAsync` is a generator function. When run, it returns an iterator object, and the iterator's `next` method returns an object with the following shape

`incrementAsync` 是一个 Generator 函数。当它运行的时候会返回一个 iterator 对象，并且该 iterator 的 `next` 方法会返回一个具有如下结构的对象

```javascript
gen.next() // => { done: boolean, value: any }
```

The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.

`value` 字段包含 yield 出来的表达式，即 `yield` 后面的表达式的结果值。`done` 字段表明该 Generator 已经终止还是仍然还有其它 'yield' 表达式需要执行。

In the case of `incrementAsync`, the generator yields 2 values consecutively:

对于 `incrementAsync`，它连续 yield 了 2 个值：

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`

So if we invoke the next method of the generator 3 times consecutively we get the following
results:

因此如果我们连续调用该 Generator 的 next 方法 3 次，我们将会得到如下结果：

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.

前面 2 次调用返回 yiled 表达式的结果。在第三次调用时由于后面没有更多的 yiled 因此 `done` 字段被置为 true。并且由于 `incrementAsync` Generator 没有返回任何值（即没有 `return` 语句），因此 `value` 字段被置为了 `undefined`。

So now, in order to test the logic inside `incrementAsync`, we'll have to iterate
over the returned Generator and check the values yielded by the generator.

所以现在为了测试 `incrementAsync` 的内部逻辑，我们不得不对 Generator 的返回值进行迭代并对它每次 yield 返回值的 value 字段进行检查。

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next(),
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

The issue is how do we test the return value of `delay`? We can't do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been easier to test.

现在的问题是我们如何测试 `delay` 的返回值？我们并不能对 Promise 做简单的等价测试。如果 `delay` 返回一个 *普通* 值（如基本数据类型的值），事情就变得简单多了。

Well, `redux-saga` provides a way to make the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly* and export it
to make a subsequent deep comparison possible:

没错，`redux-saga` 就提供了一种方式可以很优雅的解决上面问题。为了取代直接在 `incrementAsync` 中调用 `delay(1000)` 的做法，我们通过 *间接* 调用 `delay(1000)` 同时把它导出给外面以便可以做接下来的深比较（此处的“深”与深拷贝的“深”一个意思）：

```javascript
import { put, takeEvery, all, call } from 'redux-saga/effects'

export const delay = (ms) => new Promise(res => setTimeout(res, ms))

// ...

export function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)`. What's the difference?

我们现在使用 `yield call(delay, 1000)` 的方式来取代 `yield delay(1000)`。有什么不同？

In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next` (the caller could be the middleware when running our code. It could also be our test code which runs the Generator function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code above.

在第一种情况下，yield 后的表达式 `delay(1000)` 会在它被传递给 `next` 的调用者之前被执行（当运行我们的代码时，调用者可能是中间件，也可能是我们用于执行 Generator 函数并迭代其返回值的测试代码）。因此调用者得到的是一个 Promise，就像上面测试代码中的那样。

In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call` just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments. In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they return plain JavaScript objects.

在第二种情况下，将 yield 后的表达式 `call(delay, 1000)` 的返回值传递给 `next` 的调用者。`call` 和 `put` 一样，它也返回一个能够指示中间件使用指定的参数来调用指定的方法的 Effect。实际上，不管是 `put` 还是 `call` 他们本身都不执行任何 dispatch 或者异步调用，它们只是返回了一个纯 JavaScript 对象。

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

What happens is that the middleware examines the type of each yielded Effect then decides how to fulfill that Effect. If the Effect type is a `PUT` then it will dispatch an action to the Store. If the Effect is a `CALL` then it'll call the given function.

中间件会检查每个通过 yield 返回的 Effect 的类型然后决定如何来完成这个 Effect。如果 Effect 类型是 `PUT` 则它会向 Store dispatch 一个 action。如果 Effect 的类型是 `CALL` 则它会调用指定的函数。

This separation between Effect creation and Effect execution makes it possible to test our Generator in a surprisingly easy way:

这种将 Effect 创建和执行分离的做法使得我们对 Generator 代码的测试变得异常简单：

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { incrementAsync, delay } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  assert.deepEqual(
    gen.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  assert.end()
});
```

Since `put` and `call` return plain objects, we can reuse the same functions in our test code. And to test the logic of `incrementAsync`, we iterate over the generator and do `deepEqual` tests on its values.

由于 `put` 和 `call` 返回的都是纯对象，我们可以在我们的测试代码中复用相同的函数。并且为了测试 `incrementAsync` 的逻辑，我们对 Generator 进行迭代并对它的值执行 `deepEqual` 测试。

In order to run the above test, run:

运行下面的命令来执行测试：

```sh
$ npm test
```

which should report the results on the console.

它应该在控制台打印执行结果。