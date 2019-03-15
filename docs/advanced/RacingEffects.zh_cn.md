## Starting a race between multiple Effects

Sometimes we start multiple tasks in parallel but we don't want to wait for all of them, we just need
to get the *winner*: the first one that resolves (or rejects). The `race` Effect offers a way of
triggering a race between multiple Effects.

有时候我们希望并行启动多个任务但不用等所有都执行完成，只需要得到其中的 *优胜者*：即第一个 resolves（或者 rejects）。`race` Effect 提供了一种方式可以触发多个 Effects 之间的这种竞争。

The following sample shows a task that triggers a remote fetch request, and constrains the response within a
1 second timeout.

下面的示例展示了一个发出远程请求并将超时时间限定为 1 秒的 task。

```javascript
import { race, call, put, delay } from 'redux-saga/effects'

function* fetchPostsWithTimeout() {
  const {posts, timeout} = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: delay(1000)
  })

  if (posts)
    yield put({type: 'POSTS_RECEIVED', posts})
  else
    yield put({type: 'TIMEOUT_ERROR'})
}
```

Another useful feature of `race` is that it automatically cancels the loser Effects. For example,
suppose we have 2 UI buttons:

`race` 另一个有用的特性是它能够自动把输掉（即收到第一个返回之后的）的 Effects 都取消。例如，假设我们有两个 UI 按钮：

- The first starts a task in the background that runs in an endless loop `while (true)`
(e.g. syncing some data with the server each x seconds).
- 第一个按钮用于在后台启动一个运行在 `while(true)` 中的任务 （即每 x 秒向服务器同步一次数据）

- Once the background task is started, we enable a second button which will cancel the task
- 一旦后台任务启动，我们就启用第二个按钮，它能够取消该任务 


```javascript
import { race, take, call } from 'redux-saga/effects'

function* backgroundTask() {
  while (true) { ... }
}

function* watchStartBackgroundTask() {
  while (true) {
    yield take('START_BACKGROUND_TASK')
    yield race({
      task: call(backgroundTask),
      cancel: take('CANCEL_TASK')
    })
  }
}
```

In the case a `CANCEL_TASK` action is dispatched, the `race` Effect will automatically cancel
`backgroundTask` by throwing a cancellation error inside it.

如果此时 dispatch 一个 `CANCEL_TASK` action，则 `race` Effect 会通过抛出一个 canellation（取消）error 自动取消 `backgroundTask`。