# Concurrency

In the basics section, we saw how to use the helper effects `takeEvery` and `takeLatest` in order to manage concurrency between Effects.

在基础部分，我们看到了如何使用 `takeEvery` 和 `takeLatest` 来管理 Effect 之间的并发。

In this section we'll see how those helpers could be implemented using the low-level Effects.

在本节中我们将看到如何使用低级 Effect 来实现这些 helper。 

## `takeEvery`

```javascript
import {fork, take} from "redux-saga/effects"

const takeEvery = (pattern, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(pattern)
    yield fork(saga, ...args.concat(action))
  }
})
```

`takeEvery` allows multiple `saga` tasks to be forked concurrently.

`takeEvery` 允许多个 `saga` 任务并发执行。

## `takeLatest`

```javascript
import {cancel, fork, take} from "redux-saga/effects"

const takeLatest = (pattern, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(pattern)
    if (lastTask) {
      yield cancel(lastTask) // cancel is no-op if the task has already terminated
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

`takeLatest` doesn't allow multiple Saga tasks to be fired concurrently. As soon as it gets a new dispatched action, it cancels any previously-forked task (if still running).

`takeLatest` 不允许多个 Saga 任务并行执行。只要它得到一个新的 action，它就会取消之前的任何分叉任务（如果任务仍然处于运行中）。

`takeLatest` can be useful to handle AJAX requests where we want to only have the response to the latest request.

在当我们只希望获得最后一个 AJAX 请求的响应时，`takeLatest` 会非常有用。