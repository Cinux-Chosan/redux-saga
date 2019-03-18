# Pulling future actions

Until now we've used the helper effect `takeEvery` in order to spawn a new task on each incoming action. This mimics somewhat the behavior of `redux-thunk`: each time a Component, for example, invokes a `fetchProducts` Action Creator, the Action Creator will dispatch a thunk to execute the control flow.

到目前为止我们已经使用过 `takeEvery` 在每次 action 到来之时创建一个新的任务。这在一定程度上模拟了 `redux-thunk`：每当有组件调用 `fetchProducts` Action 生成器，生成器就会 dispatch 一个 thunk （指`redux-thunk`中的thunk，和 `redux-saga`中的saga类似）来执行控制流。

In reality, `takeEvery` is just a wrapper effect for internal helper function built on top of the lower-level and more powerful API. In this section we'll see a new Effect, `take`, which makes it possible to build complex control flow by allowing total control of the action observation process.

实际上，`takeEvery` 只是一个基于那些强大的底层 API 的一个封装。在本章节中我们将看到一个新的 Effect —— `take`，通过它对 action 观测过程的全面控制，使得我们可以建立复杂的控制流。


## A basic logger

Let's take a basic example of a Saga that watches all actions dispatched to the store and logs them to the console.

让我们从一个监视所有被 dispatch 到 store 的 actions 并把它们打印到控制台的基本 Saga 示例开始。

Using `takeEvery('*')` (with the wildcard `*` pattern), we can catch all dispatched actions regardless of their types.

使用 `takeEvery('*')` 可以捕获所有 actions，不管它们的 type 是什么。

```javascript
import { select, takeEvery } from 'redux-saga/effects'

function* watchAndLog() {
  yield takeEvery('*', function* logger(action) {
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  })
}
```

Now let's see how to use the `take` Effect to implement the same flow as above:

现在来看看如何使用 `take` 实现上面相同的效果：

```javascript
import { select, take } from 'redux-saga/effects'

function* watchAndLog() {
  while (true) {
    const action = yield take('*')
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  }
}
```

The `take` is just like `call` and `put` we saw earlier. It creates another command object that tells the middleware to wait for a specific action. The resulting behavior of the `call` Effect is the same as when the middleware suspends the Generator until a Promise resolves. In the `take` case, it'll suspend the Generator until a matching action is dispatched. In the above example, `watchAndLog` is suspended until any action is dispatched.

`take` 和我们之前看到的 `call` 和 `put` 一样。它创建了一个指令对象告诉中间件需要等待某个特定的 action。其效果和 `call` 会让中间件将 Generator 挂起直到 Promise 被 resolve 一样，在使用 `take` 的情况下，它会将 Generator 挂起直到匹配的 action 被 dispatched。在上面的例子中， `watchAndLog` 会被挂起直到任意 action 被 dispatched。

Note how we're running an endless loop `while (true)`. Remember, this is a Generator function, which doesn't have a run-to-completion behavior. Our Generator will block on each iteration waiting for an action to happen.

注意我们使用了无限循环 `while(true)`。记住，这是 Generator 函数，它不会有 run-to-completion（理解为一般函数的从头到尾一次执行完成）的行为。我们这里的 Generator 会在每次迭代中阻塞以等待 action 发生。

Using `take` has a subtle impact on how we write our code. In the case of `takeEvery`, the invoked tasks have no control on when they'll be called. They will be invoked again and again on each matching action. They also have no control on when to stop the observation.

使用 `take` 对我们写代码的方式有细微的影响。使用 `takeEvery` 的时候，如果 task 一经调用，我们无法对它的调用时机进行控制（个人理解：比如 `takeEvery` 一经执行，就不能像 `take` 那样配合流程控制语句进行选择什么条件下需要执行）。它们会在收到匹配的 action 时一遍又一遍的被调用。并且也不能控制它们何时停止对 action 的监视。

In the case of `take`, the control is inverted. Instead of the actions being *pushed* to the handler tasks, the Saga is *pulling* the action by itself. It looks as if the Saga is performing a normal function call `action = getNextAction()` which will resolve when the action is dispatched.

而是用 `take` 的话，控制就被反转了。Saga 主动 *拉取* action，而不是被动的接收 *推送* 过来的 action 给 task 处理器。看起来就像 Saga 执行了一个普通函数 `action = getNextAction()`，这个函数会等到有 action 被 dispatch 过来时变为 resolve 状态。

This inversion of control allows us to implement control flows that are non-trivial to do with the traditional *push* approach.

这种控制反转使得我们可以实现与传统 *push* 方式不同的控制流。

As a basic example, suppose that in our Todo application, we want to watch user actions and show a congratulation message after the user has created his first three todos.

作为基本示例，假如在我们的待办事项应用中，我们希望观察用户操作并在用户首次创建 3 条待办事项后显示一条祝贺消息：

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

Instead of a `while (true)`, we're running a `for` loop, which will iterate only three times. After taking the first three `TODO_CREATED` actions, `watchFirstThreeTodosCreation` will cause the application to display a congratulation message then terminate. This means the Generator will be garbage collected and no more observation will take place.

我们这里使用了只会迭代 3 次的 `for` 循环来代替 `while(true)`。在收到前三个 `TODO_CREATED` actions 之后，`watchFirstThreeTodosCreation` 会显示一条祝贺消息然后退出。这意味着 Generator 会被垃圾回收并且不会再继续监视。

Another benefit of the pull approach is that we can describe our control flow using a familiar synchronous style. For example, suppose we want to implement a login flow with two actions: `LOGIN` and `LOGOUT`. Using `takeEvery` (or `redux-thunk`), we'll have to write two separate tasks (or thunks): one for `LOGIN` and the other for `LOGOUT`.

这种 pull 的方式带来的另一个好处就是我们可以使用熟悉的同步编码方式来描述我们的控制流。例如，假设我们希望实现具有两个 actions 的登录操作流，分别是 `LOGIN` 和 `LOGOUT`。使用 `takeEvery`（或者 `redux-thunk`），我们将不得不写两个独立的 tasks （或者 thunks）：一个用于 `LOGIN` 另一个用与 `LOGOUT`。

The result is that our logic is now spread in two places. In order for someone reading our code to understand it, they would have to read the source of the two handlers and make the link between the logic in both in their head. In other words, it means they would have to rebuild the model of the flow in their head by rearranging mentally the logic placed in various places of the code in the correct order.

但这样做的结果就是我们的逻辑被分到两个地方。对于那些要从我们代码中读懂登录逻辑的人来说，他们不得不从两个地方读取源代码并把它们的逻辑在大脑中联系起来。换句话说，这意味着他们不得不在思维上将不同位置的代码逻辑重新排列以便在头脑中重建出正确的登录流程模型。

Using the pull model, we can write our flow in the same place instead of handling the same action repeatedly.

而用 pull 模型时，我们可以将我们的流程写在同一个地方，而不用对同一个 action 分开处理。

```javascript
function* loginFlow() {
  while (true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}
```

The `loginFlow` Saga more clearly conveys the expected action sequence. It knows that the `LOGIN` action should always be followed by a `LOGOUT` action, and that `LOGOUT` is always followed by a `LOGIN` (a good UI should always enforce a consistent order of the actions, by hiding or disabling unexpected actions).

`loginFlow` Saga 更清楚的展示了我们期望的 action 顺序。`LOGIN` action 后面始终应该是 `LOGOUT`，`LOGOUT` 之后又肯定是 `LOGIN`（一个好的用户界面应该总是通过隐藏或禁用非预期 action 来强制执行一致的 action 顺序）