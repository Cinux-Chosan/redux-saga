# Non-blocking calls

In the previous section, we saw how the `take` Effect allows us to better describe a non-trivial flow in a central place.

在之前的章节中，我们看到 `take` Effect 是如何让我们可以用一个中心位置来描述一个非凡的流。

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

现在我们来完善这个示例，实现真实的 登录/注销 逻辑。假设我们有一个 API 用于从远程服务器验证对用户身份进行验证。如果验证成功，则服务器会返回一个 token，我们通过 DOM storage 将这个 token 保存在应用中（假设我们的 API 提供了一个 DOM storage 服务）。

When the user logs out, we'll delete the authorization token stored previously.

当用户注销时，我们就将之前存下来的 token 删除。

### First try

So far we have all Effects needed to implement the above flow. We can wait for specific actions in the store using the `take` Effect. We can make asynchronous calls using the `call` Effect. Finally, we can dispatch actions to the store using the `put` Effect.

目前为止我们已经具有所有用于实现上面流程的 Effects。我们可以使用 `take` 来等待特定 actions 被发送到 store。我们可以使用 `call` 来做异步调用。最后，我们还可以使用 `put` 来向 store dispatch action。

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

`loginFlow` 在 `while(true)` 内部实现了整个流程，一旦到达流程的最后一步 (`LOGOUT`) ，我们又会在下一个循环中等待 `LOGIN_REQUEST` action。

`loginFlow` first waits for a `LOGIN_REQUEST` action. Then, it retrieves the credentials in the action payload (`user` and `password`) and makes a `call` to the `authorize` task.

`loginFlow` 首先等待 `LOGIN_REQUEST` action。然后，它会检索 action 中的用户凭证（`user` 和 `password`）并用 `call` 发起一个对 `authorize` 任务的调用。

As you noted, `call` isn't only for invoking functions returning Promises. We can also use it to invoke other Generator functions. In the above example, **`loginFlow` will wait for authorize until it terminates and returns** (i.e. after performing the api call, dispatching the action and then returning the token to `loginFlow`).

如你所见， `call` 不仅可以用于调用返回 Promises 的函数。我们还可以用它来调用其他 Generator 函数。在上面的示例中，**`loginFlow` 会等待 authorize 直到它运行结束并返回**（即在执行 api 调用之后，dispatch 一个 action 然后将 token 返回给 `loginFlow`）。

If the API call succeeds, `authorize` will dispatch a `LOGIN_SUCCESS` action then return the fetched token. If it results in an error, it'll dispatch a `LOGIN_ERROR` action.

如果 API 调用成功， `authorize` 将会 dispatch 一个 `LOGIN_SUCCESS` action 然后返回获取到的 token。如果导致错误发生，则 dispatch 一个 `LOGIN_ERROR` action。

If the call to `authorize` is successful, `loginFlow` will store the returned token in the DOM storage and wait for a `LOGOUT` action. When the user logs out, we remove the stored token and wait for a new user login.

如果 `authorize` 调用成功，`loginFlow` 会将得到的 token 存储在 DOM storage 并开始等待 `LOGOUT` action 发生。当用户注销时，清除 token 并等待新一轮用户登录。

If the `authorize` failed, it'll return `undefined`, which will cause `loginFlow` to skip the previous process and wait for a new `LOGIN_REQUEST` action.

如果 `authorize` 失败，则返回 `undefined`，这会使 `loginFlow` 直接跳过之后的流程并等待新一轮 `LOGIN_REQUEST` action。

Observe how the entire logic is stored in one place. A new developer reading our code doesn't have to travel between various places to understand the control flow. It's like reading a synchronous algorithm: steps are laid out in their natural order. And we have functions which call other functions and wait for their results.

整个逻辑存放在一个地方，这样新来的开发者在阅读我们的代码时就不需要在多个地方之间来回切换。就像阅读同步代码一样：每个步骤都按照代码的自然顺序发生。并且我们有的函数需要调用其它函数并等待其执行结果。

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

用户点击了 `Logout` 按钮从而 dispatch 了一个 `LOGOUT` action。

The following example illustrates the hypothetical sequence of the events:

我们用以下示例来说明事件的发生顺序：

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

The problem with the above code is that `call` is a blocking Effect. i.e. the Generator can't perform/handle anything else until the call terminates. But in our case we do not only want `loginFlow` to execute the authorization call, but also watch for an eventual `LOGOUT` action that may occur in the middle of this call. That's because `LOGOUT` is *concurrent* to the `authorize` call.

So what's needed is some way to start `authorize` without blocking so `loginFlow` can continue and watch for an eventual/concurrent `LOGOUT` action.

To express non-blocking calls, the library provides another Effect: [`fork`](https://redux-saga.js.org/docs/api/index.html#forkfn-args). When we fork a *task*, the task is started in the background and the caller can continue its flow without waiting for the forked task to terminate.

So in order for `loginFlow` to not miss a concurrent `LOGOUT`, we must not `call` the `authorize` task, instead we have to `fork` it.

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

- If the `authorize` task succeeds before the user logs out, it'll dispatch a `LOGIN_SUCCESS` action, then terminate. Our `loginFlow` saga will then wait only for a future `LOGOUT` action (because `LOGIN_ERROR` will never happen).

- If the `authorize` fails before the user logs out, it will dispatch a `LOGIN_ERROR` action, then terminate. So `loginFlow` will take the `LOGIN_ERROR` before the `LOGOUT` then it will enter in a another `while` iteration and will wait for the next `LOGIN_REQUEST` action.

- If the user logs out before the `authorize` terminates, then `loginFlow` will take a `LOGOUT` action and also wait for the next `LOGIN_REQUEST`.

Note the call for `Api.clearItem` is supposed to be idempotent. It'll have no effect if no token was stored by the `authorize` call. `loginFlow` makes sure no token will be in the storage before waiting for the next login.

But we're not yet done. If we take a `LOGOUT` in the middle of an API call, we have to **cancel** the `authorize` process, otherwise we'll have 2 concurrent tasks evolving in parallel: The `authorize` task will continue running and upon a successful (resp. failed) result, will dispatch a `LOGIN_SUCCESS` (resp. a `LOGIN_ERROR`) action leading to an inconsistent state.

In order to cancel a forked task, we use a dedicated Effect [`cancel`](https://redux-saga.js.org/docs/api/index.html#canceltask)

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

We are *almost* done (concurrency is not that easy; you have to take it seriously).

Suppose that when we receive a `LOGIN_REQUEST` action, our reducer sets some `isLoginPending` flag to true so it can display some message or spinner in the UI. If we get a `LOGOUT` in the middle of an API call and abort the task by *killing it* (i.e. the task is stopped right away), then we may end up again with an inconsistent state. We'll still have `isLoginPending` set to true and our reducer will be waiting for an outcome action (`LOGIN_SUCCESS` or `LOGIN_ERROR`).

Fortunately, the `cancel` Effect won't brutally kill our `authorize` task. Instead, it'll give it a chance to perform its cleanup logic. The cancelled task can handle any cancellation logic (as well as any other type of completion) in its `finally` block. Since a finally block execute on any type of completion (normal return, error, or forced cancellation), there is an Effect `cancelled` which you can use if you want handle cancellation in a special way:

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

- dispatch a dedicated action `RESET_LOGIN_PENDING`
- make the reducer clear the `isLoginPending` on a `LOGOUT` action
