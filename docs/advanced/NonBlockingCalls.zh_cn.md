# Non-blocking calls

In the previous section, we saw how the `take` Effect allows us to better describe a non-trivial flow in a central place.

在之前的章节中，我们看到了如何用 `take` Effect 将一个完整的流程放在一起。

Revisiting the login flow example:

再次回顾一下登录流程示例：

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

Let's complete the example and implement the actual login/logout logic. Suppose we have an API which permits us to authorize the user on a remote server. If the authorization is successful, the server will return an authorization token which will be stored by our application using DOM storage (assume our API provides another service for DOM storage).

现在我们来完善这个示例，实现真实的 登录/注销 逻辑。假设我们有一个 API 用于从远程服务器对用户身份进行验证。如果验证成功，则服务器会返回一个 token，我们通过 DOM storage 将这个 token 保存在应用中（假设我们的 API 中还提供了一个 DOM storage 服务）。

When the user logs out, we'll delete the authorization token stored previously.

当用户注销时，我们就将之前存下来的 token 清除。

### First try

So far we have all Effects needed to implement the above flow. We can wait for specific actions in the store using the `take` Effect. We can make asynchronous calls using the `call` Effect. Finally, we can dispatch actions to the store using the `put` Effect.

目前为止我们已经具有实现上面流程所需的全部 Effects。我们可以使用 `take` Effect 来等待 store 中某些特定的 actions。我们可以使用 `call` Effect 来做异步调用。最后，我们还可以使用 `put` Effect 来向 store dispatch action。

Let's give it a try:

我们来试一试：

> Note: the code below has a subtle issue. Make sure to read the section until the end.
>
> 注意：下面的代码有一点小问题。请务必将本章节读完。

```javascript
import { take, call, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    const token = yield call(authorize, user, password)
    if (token) {
      yield call(Api.storeItem, {token})
      yield take('LOGOUT')
      yield call(Api.clearItem, 'token')
    }
  }
}
```

First, we created a separate Generator `authorize` which will perform the actual API call and notify the Store upon success.

首先，我们创建了一个独立的 Generator `authorize` 用于执行实际的 API 调用并在成功后通知 Store。

The `loginFlow` implements its entire flow inside a `while (true)` loop, which means once we reach the last step in the flow (`LOGOUT`) we start a new iteration by waiting for a new `LOGIN_REQUEST` action.

`loginFlow` 在 `while(true)` 内部实现了整个流程，一旦到达流程的最后一步 (`LOGOUT`) ，我们又会启动新一轮的循环来等待新的 `LOGIN_REQUEST` action。

`loginFlow` first waits for a `LOGIN_REQUEST` action. Then, it retrieves the credentials in the action payload (`user` and `password`) and makes a `call` to the `authorize` task.

`loginFlow` 首先等待 `LOGIN_REQUEST` action。然后，它会从 action 中检索出用户凭证（`user` 和 `password`）并用 `call` 发起一个对 `authorize` 任务的调用。

As you noted, `call` isn't only for invoking functions returning Promises. We can also use it to invoke other Generator functions. In the above example, **`loginFlow` will wait for authorize until it terminates and returns** (i.e. after performing the api call, dispatching the action and then returning the token to `loginFlow`).

如你所见， `call` 不仅可以用于调用返回 Promises 的函数。我们还可以用它来调用其他 Generator 函数。在上面的示例中，**`loginFlow` 会等待 authorize 直到它运行结束并返回**（即在执行 api 调用之后，dispatch 一个 action 然后将 token 返回给 `loginFlow`）。

If the API call succeeds, `authorize` will dispatch a `LOGIN_SUCCESS` action then return the fetched token. If it results in an error, it'll dispatch a `LOGIN_ERROR` action.

如果 API 调用成功， `authorize` 将会 dispatch 一个 `LOGIN_SUCCESS` action 然后将获取到的 token 返回。如果导致错误发生，则 dispatch 一个 `LOGIN_ERROR` action。

If the call to `authorize` is successful, `loginFlow` will store the returned token in the DOM storage and wait for a `LOGOUT` action. When the user logs out, we remove the stored token and wait for a new user login.

如果 `authorize` 调用成功，`loginFlow` 会将得到的 token 存储在 DOM storage 并开始等待 `LOGOUT` action 发生。当用户注销时，我们清除 token 并等待新一轮用户登录。

If the `authorize` failed, it'll return `undefined`, which will cause `loginFlow` to skip the previous process and wait for a new `LOGIN_REQUEST` action.

如果 `authorize` 失败，它会返回 `undefined`，这会使 `loginFlow` 直接跳过之后的流程并等待新一轮 `LOGIN_REQUEST` action。

Observe how the entire logic is stored in one place. A new developer reading our code doesn't have to travel between various places to understand the control flow. It's like reading a synchronous algorithm: steps are laid out in their natural order. And we have functions which call other functions and wait for their results.

整个逻辑存放在一个地方，这样新来的开发者在阅读我们的代码时就不至于在多个地方之间来回切换。就像阅读同步代码一样：每个步骤都按照代码的自然顺序发生。即便有的函数需要调用其它函数并等待其执行结果。

### But there is still a subtle issue with the above approach（但上面的代码还是有一点小问题）

Suppose that when the `loginFlow` is waiting for the following call to resolve:

假设 `loginFlow` 一直在等待如下调用：

```javascript
function* loginFlow() {
  while (true) {
    // ...
    try {
      const token = yield call(authorize, user, password)
      // ...
    }
    // ...
  }
}
```

The user clicks on the `Logout` button causing a `LOGOUT` action to be dispatched.

此时用户点击了 `Logout` 按钮从而触发了一个 `LOGOUT` action。

The following example illustrates the hypothetical sequence of the events:

我们用以下示例来描述事件发生的假想顺序：

```
UI                              loginFlow
--------------------------------------------------------
LOGIN_REQUEST...................call authorize.......... waiting to resolve
........................................................
........................................................
LOGOUT.................................................. missed!
........................................................
................................authorize returned...... dispatch a `LOGIN_SUCCESS`!!
........................................................
```

When `loginFlow` is blocked on the `authorize` call, an eventual `LOGOUT` occurring in between the call and the response will be missed, because `loginFlow` hasn't yet performed the `yield take('LOGOUT')`.

当 `loginFlow` 被 `authorize` 调用阻塞时，在开始调用和收到响应之间触发的 `LOGOUT` 将会被 `loginFlow` 错过，因为 `loginFlow` 还没有执行到 `yield take('LOGOUT')`。

The problem with the above code is that `call` is a blocking Effect. i.e. the Generator can't perform/handle anything else until the call terminates. But in our case we do not only want `loginFlow` to execute the authorization call, but also watch for an eventual `LOGOUT` action that may occur in the middle of this call. That's because `LOGOUT` is *concurrent* to the `authorize` call.

上面代码中的问题就是由于 `call` 是一个阻塞 Effect 导致的。即 Generator 不会在 call 执行完成前执行和处理任何事情。但我们的场景是希望 `loginFlow` 既要执行 `authorize` 调用同时又要监听中途发生的 `LOGOUT` action。因为 `LOGOUT` 和 `authorize` 调用应该是 *并行* 的。

So what's needed is some way to start `authorize` without blocking so `loginFlow` can continue and watch for an eventual/concurrent `LOGOUT` action.

因此我们需要一种能启动 `authorize` 同时又不会阻塞 `loginFlow` 的方式以便 `loginFlow` 能够继续监听可能并行出现的 `LOGOUT` action。

To express non-blocking calls, the library provides another Effect: [`fork`](https://redux-saga.js.org/docs/api/index.html#forkfn-args). When we fork a *task*, the task is started in the background and the caller can continue its flow without waiting for the forked task to terminate.

为了能够执行非阻塞调用，`redux-saga` 提供了另一个 Effect：[`fork`](https://redux-saga.js.org/docs/api/index.html#forkfn-args)。当我们 fork 一个 *task（任务）* 时，task 会在后台启动，因此调用者不用等待该 task 运行结束从而可以继续执行它后面的流程。

So in order for `loginFlow` to not miss a concurrent `LOGOUT`, we must not `call` the `authorize` task, instead we have to `fork` it.

因此为了让 `loginFlow` 不错过并行产生的 `LOGOUT`，我们需要 `fork` 一个 `authorize` task 而不是 `call` 它。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'

function* loginFlow() {
  while (true) {
    ...
    try {
      // non-blocking call, what's the returned value here ?
      const ?? = yield fork(authorize, user, password)
      ...
    }
    ...
  }
}
```

The issue now is since our `authorize` action is started in the background, we can't get the `token` result (because we'd have to wait for it). So we need to move the token storage operation into the `authorize` task.

现在的问题是由于 `authorize` action 是在后台启动，我们拿不到它返回的 `token`（因为我们没有等待它执行完成）。因此我们需要把 token 存储操作移到 `authorize` 任务中去。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    yield fork(authorize, user, password)
    yield take(['LOGOUT', 'LOGIN_ERROR'])
    yield call(Api.clearItem, 'token')
  }
}
```

We're also doing `yield take(['LOGOUT', 'LOGIN_ERROR'])`. It means we are watching for 2 concurrent actions:

我们这里用了 `yield take(['LOGOUT', 'LOGIN_ERROR'])`。它表示我们正在同时监视两个并行的 actions：

- If the `authorize` task succeeds before the user logs out, it'll dispatch a `LOGIN_SUCCESS` action, then terminate. Our `loginFlow` saga will then wait only for a future `LOGOUT` action (because `LOGIN_ERROR` will never happen).
- 如果 `authorize` 任务在用户注销之前执行成功，它会 dispatch 一个 `LOGIN_SUCCESS` action，然后结束。`loginFlow` saga 将只会等待接下来的 `LOGOUT` action（因为这种情况下 `LOGIN_ERROR` 永远不会发生）。

- If the `authorize` fails before the user logs out, it will dispatch a `LOGIN_ERROR` action, then terminate. So `loginFlow` will take the `LOGIN_ERROR` before the `LOGOUT` then it will enter in a another `while` iteration and will wait for the next `LOGIN_REQUEST` action.
- 如果 `authorize` 在用户注销前失败，它会 dispatch 一个 `LOGIN_ERROR` action，然后结束。因此 `loginFlow` 将会在收到 `LOGOUT` 之前收到 `LOGIN_ERROR` 然后进入另一个 `while` 循环并等待下一个 `LOGIN_REQUEST` action。

- If the user logs out before the `authorize` terminates, then `loginFlow` will take a `LOGOUT` action and also wait for the next `LOGIN_REQUEST`.
- 如果用户在 `authorize` 终止前注销，则 `loginFlow` 会接收到 `LOGOUT` action 并等待下一个 `LOGIN_REQUEST`。

Note the call for `Api.clearItem` is supposed to be idempotent. It'll have no effect if no token was stored by the `authorize` call. `loginFlow` makes sure no token will be in the storage before waiting for the next login.

注意对 `Api.clearItem` 的调用应该是幂等的。如果 `authorize` 没有存储 token 则它不应该有任何效果。`loginFlow` 应该确保在等待下一个 login 之前 storage 中没有 token 存在。

But we're not yet done. If we take a `LOGOUT` in the middle of an API call, we have to **cancel** the `authorize` process, otherwise we'll have 2 concurrent tasks evolving in parallel: The `authorize` task will continue running and upon a successful (resp. failed) result, will dispatch a `LOGIN_SUCCESS` (resp. a `LOGIN_ERROR`) action leading to an inconsistent state.

但现在我们还没有完成。如果在 API 调用中途收到一个 `LOGOUT` ，我们不得不 **取消** `authorize` 进程，否则我们就有了两个任务在并行执行：`authorize` 任务将继续执行，一旦执行成功（也可能是失败），它将会 dispatch 一个 `LOGIN_SUCCESS`（如果失败就是 `LOGIN_ERROR`） action 从而导致状态不一致。

In order to cancel a forked task, we use a dedicated Effect [`cancel`](https://redux-saga.js.org/docs/api/index.html#canceltask)

为了能够取消 `fork` 出的 task，我们需要使用一个专用 Effect [`cancel`](https://redux-saga.js.org/docs/api/index.html#canceltask)

```javascript
import { take, put, call, fork, cancel } from 'redux-saga/effects'

// ...

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    // fork return a Task object
    const task = yield fork(authorize, user, password)
    const action = yield take(['LOGOUT', 'LOGIN_ERROR'])
    if (action.type === 'LOGOUT')
      yield cancel(task)
    yield call(Api.clearItem, 'token')
  }
}
```

`yield fork` results in a [Task Object](https://redux-saga.js.org/docs/api/index.html#task). We assign the returned object into a local constant `task`. Later if we take a `LOGOUT` action, we pass that task to the `cancel` Effect. If the task is still running, it'll be aborted. If the task has already completed then nothing will happen and the cancellation will result in a no-op. And finally, if the task completed with an error, then we do nothing, because we know the task already completed.

`yield fork` 会得到一个 [Task 对象](https://redux-saga.js.org/docs/api/index.html#task)。我们将该 task 对象赋值给局部常量 `task`。稍后如果我们收到 `LOGOUT` action，我们需要将它传递给 `cacel` Effect。如果此时 task 仍然在运行，则它会被终止。如果 task 已经执行完成，则取消操作（`cancel`）不做任何事情。最后，如果 task 出现错误，我们也不做任何操作，因为我们知道 task 已经完成。

We are *almost* done (concurrency is not that easy; you have to take it seriously).

我们快要完成了（并发并没有想象中的那么容易，你必须认真对待它）。

Suppose that when we receive a `LOGIN_REQUEST` action, our reducer sets some `isLoginPending` flag to true so it can display some message or spinner in the UI. If we get a `LOGOUT` in the middle of an API call and abort the task by *killing it* (i.e. the task is stopped right away), then we may end up again with an inconsistent state. We'll still have `isLoginPending` set to true and our reducer will be waiting for an outcome action (`LOGIN_SUCCESS` or `LOGIN_ERROR`).

假设当我们收到一个 `LOGIN_REQUEST` action，我们在 reducer 中将 `isLoginPending` 标志设置为了 true，它会告诉 UI 渲染等待加载提示。如果在 API 调用期间又收到了 `LOGOUT` 并通过 *杀掉任务* （即让 task 立即停止）的方式退出任务，则又可能出现状态不一致的情况。此时 `isLoginPending` 仍然被设置为了 true 但是 reducer 却在等待登录结果的 action(`LOGIN_SUCCESS` 或者 `LOGIN_ERROR`)。

Fortunately, the `cancel` Effect won't brutally kill our `authorize` task. Instead, it'll give it a chance to perform its cleanup logic. The cancelled task can handle any cancellation logic (as well as any other type of completion) in its `finally` block. Since a finally block execute on any type of completion (normal return, error, or forced cancellation), there is an Effect `cancelled` which you can use if you want handle cancellation in a special way:

幸运的是，`cancel` Effect 不会残忍地杀掉我们的 `authorize` task。取而代之的是，它会给 `authorize` 机会让它执行自己的清理逻辑。被取消的 task 能够在其 `finally` 语句块中处理任何关于取消的逻辑（包括任何其它的完成类型）. 因为 finally 语句块会在任何完成类型之上执行（正常 return，error 或者强制取消）（译者注：这里我的理解是 `finally` 在任何情况下都会执行，它不会受到 return、error 或者其他情况的影响），如果你希望以这种特别的方式来执行取消后的逻辑，你可以使用 `cancelled` Effect：

```javascript
import { take, call, put, cancelled } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  } finally {
    if (yield cancelled()) {
      // ... put special cancellation handling code here
    }
  }
}
```

You may have noticed that we haven't done anything about clearing our `isLoginPending` state. For that, there are at least two possible solutions:

你可能已经注意到了我们并没有实现任何关于清除 `isLoginPending` 状态的逻辑。但至少有两种可能的方案：

- dispatch a dedicated action `RESET_LOGIN_PENDING`
- dispatch 一个专用 action `RESET_LOGIN_PENDING`
- make the reducer clear the `isLoginPending` on a `LOGOUT` action
- 让 reducer 在 `LOGOUT` action 中清除 `isLoginPending`
