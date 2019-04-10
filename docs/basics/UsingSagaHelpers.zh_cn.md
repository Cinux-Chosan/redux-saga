# Using Saga Helpers（已校验）

`redux-saga` provides some helper effects wrapping internal functions to spawn tasks when some specific actions are dispatched to the Store.

`redux-saga` 提供了一些辅助 effect 用于当 Store 接收到指定 action 时用其（指辅助 effect）内部封装的函数来创建任务。

The helper functions are built on top of the lower level API. In the advanced section, we'll see how those functions can be implemented.

这些辅助函数都是对底层 API 的封装。在高级部分，我们将看到它们是如何被实现的。

The first function, `takeEvery` is the most familiar and provides a behavior similar to `redux-thunk`.

第一个函数，也是我们最熟悉的一个函数 —— `takeEvery` ，它的行为类似于 `redux-thunk`。

Let's illustrate with the common AJAX example. On each click on a Fetch button we dispatch a `FETCH_REQUESTED` action. We want to handle this action by launching a task that will fetch some data from the server.

我们用一个普通的 AJAX 示例来进行讲解。每当 Fetch 按钮被点击时我们就发出（dispatch）一个 `FETCH_REQUESTED` action，在收到这个 action 时就启动一个从服务器获取数据的任务。

First we create the task that will perform the asynchronous action:

首先，我们创建一个用于执行异步 action 的任务：

```javascript
import { call, put } from 'redux-saga/effects'

export function* fetchData(action) {
   try {
      const data = yield call(Api.fetchUser, action.payload.url)
      yield put({type: "FETCH_SUCCEEDED", data})
   } catch (error) {
      yield put({type: "FETCH_FAILED", error})
   }
}
```

To launch the above task on each `FETCH_REQUESTED` action:

然后在每次收到 `FETCH_REQUESTED` action 的时候就启动上面创建的任务：

```javascript
import { takeEvery } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}
```

In the above example, `takeEvery` allows multiple `fetchData` instances to be started concurrently. At a given moment, we can start a new `fetchData` task while there are still one or more previous `fetchData` tasks which have not yet terminated.

上面的例子中，`takeEvery` 允许多个 `fetchData` 实例并行启动。也就是说即使此刻前面还有多个 `fetchData` 任务没有执行完成我们依然可以启动新的 `fetchData` 任务。

If we want to only get the response of the latest request fired (e.g. to always display the latest version of data) we can use the `takeLatest` helper:

如果我们只想获取最后一个请求的响应（即总是展示最新数据）我们可以使用 `takeLatest`：

```javascript
import { takeLatest } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeLatest('FETCH_REQUESTED', fetchData)
}
```

Unlike `takeEvery`, `takeLatest` allows only one `fetchData` task to run at any moment. And it will be the latest started task. If a previous task is still running when another `fetchData` task is started, the previous task will be automatically cancelled.

和 `takeEvery` 不同，`takeLatest` 在同一时刻只能运行一个 `fetchData` 任务 —— 即最后启动的那个任务。如果在启动新的任务时前一个任务仍然在运行，则会先自动取消前一个任务。

If you have multiple Sagas watching for different actions, you can create multiple watchers with those built-in helpers, which will behave like there was `fork` used to spawn them (we'll talk about `fork` later. For now, consider it to be an Effect that allows us to start multiple sagas in the background).

如果你需要多个 Saga 来对不同的 action 进行响应，你可以使用这些内置辅助函数来创建多个观察者（watchers），其行为如同是使用 `fork` 来创建它们那样（稍后我们会谈到 `fork`。现在只需要把它当做是一个可以在后台启动多个 saga 的 Effect）

For example:

例如：

```javascript
import { takeEvery } from 'redux-saga/effects'

// FETCH_USERS
function* fetchUsers(action) { ... }

// CREATE_USER
function* createUser(action) { ... }

// use them in parallel
export default function* rootSaga() {
  yield takeEvery('FETCH_USERS', fetchUsers)
  yield takeEvery('CREATE_USER', createUser)
}
```
