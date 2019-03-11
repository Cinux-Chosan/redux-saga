# Using Saga Helpers

`redux-saga` provides some helper effects wrapping internal functions to spawn tasks when some specific actions are dispatched to the Store.

`redux-saga` 提供了一些辅助 effects，它将函数封装在内部，当某个特定的 action 被 dispatch 到 Store 时它会创建任务去执行。

The helper functions are built on top of the lower level API. In the advanced section, we'll see how those functions can be implemented.

这些辅助函数基于底层 API 之上。在高级部分，我们将看到它们是如何实现的。

The first function, `takeEvery` is the most familiar and provides a behavior similar to `redux-thunk`.

第一个函数，`takeEvery` 是最常用的，它的行为类似于 `redux-thunk`。

Let's illustrate with the common AJAX example. On each click on a Fetch button we dispatch a `FETCH_REQUESTED` action. We want to handle this action by launching a task that will fetch some data from the server.

让我们从最常用的 AJAX 例子开始进行讲解。每当点击 Fetch 按钮的时候我们就 dispatch 一条 `FETCH_REQUESTED` action。我们希望收到这个 action 的时候启动一个任务来从服务器获取数据。

First we create the task that will perform the asynchronous action:

首先，我们创建一个可以执行异步 action 的任务：

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

然后在每次收到 `FETCH_REQUESTED` action 的时候启动上面创建的任务：

```javascript
import { takeEvery } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}
```

In the above example, `takeEvery` allows multiple `fetchData` instances to be started concurrently. At a given moment, we can start a new `fetchData` task while there are still one or more previous `fetchData` tasks which have not yet terminated.

上面的例子中，`takeEvery` 允许多个 `fetchData` 实例并行启动。也就是说即使此刻前面还有多个 `fetchData` 任务没有执行完成的时候我们依然可以启动新 `fetchData` 任务。

If we want to only get the response of the latest request fired (e.g. to always display the latest version of data) we can use the `takeLatest` helper:

如果我们只想获取最后一个请求的响应（即总是展示最新数据）我们可以使用 `takeLatest`：

```javascript
import { takeLatest } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeLatest('FETCH_REQUESTED', fetchData)
}
```

Unlike `takeEvery`, `takeLatest` allows only one `fetchData` task to run at any moment. And it will be the latest started task. If a previous task is still running when another `fetchData` task is started, the previous task will be automatically cancelled.

和 `takeEvery` 不同，`takeLatest` 在同一时刻只能运行一个 `fetchData` 任务 —— 即最后启动的那一个任务。如果在启动新的任务时前一个任务仍然在运行，则前一个任务会被自动取消。

If you have multiple Sagas watching for different actions, you can create multiple watchers with those built-in helpers, which will behave like there was `fork` used to spawn them (we'll talk about `fork` later. For now, consider it to be an Effect that allows us to start multiple sagas in the background).

如果你需要多个 Sagas 响应不同的 actions，你可以使用这些内置辅助函数创建多个监视器（watchers），它们的行为就像是使用了 `fork` 来创建它们（稍后我们会谈到 `fork`。现在只需要把它当做是一个可以允许我们在后台启动多个 sagas 的 Effect）

For example:

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