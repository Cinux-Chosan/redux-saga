# Glossary

This is a glossary of the core terms in Redux Saga.

这是 Redux Saga 中核心术语的词汇表。

### Effect

An effect is a plain JavaScript Object containing some instructions to be executed by the saga middleware.

一个 effect 就是一个纯 JavaScript 对象，它包含了需要被 saga 中间件执行的指令信息。

You create effects using factory functions provided by the redux-saga library. For example you use
`call(myfunc, 'arg1', 'arg2')` to instruct the middleware to invoke `myfunc('arg1', 'arg2')` and return
the result back to the Generator that yielded the effect

你需要使用 `redux-saga` 通过的工厂函数来创建 effect。例如使用 `call(myfunc, 'arg1', 'arg2')` 来告诉中间件执行 `myfunc('arg1', 'arg2')` 并把结果返回给 Generator。

### Task

A task is like a process running in background. In a redux-saga based application there can be
multiple tasks running in parallel. You create tasks by using the `fork` function

task 就像是运行在后台的进程一样。在基于 redux-saga 的应用中，多个 tasks 可以并行运行。使用 `fork` 来创建 tasks。

```javascript
import {fork} from "redux-saga/effects"

function* saga() {
  ...
  const task = yield fork(otherSaga, ...args)
  ...
}
```

### Blocking/Non-blocking call

A Blocking call means that the Saga yielded an Effect and will wait for the outcome of its execution before
resuming to the next instruction inside the yielding Generator.

阻塞调用指 Saga 抛出一个 Effect 后并在恢复 Generator 之前等待执行结果。

A Non-blocking call means that the Saga will resume immediately after yielding the Effect.

非阻塞调用指 Saga 在抛出 Effect 之后会立即恢复。

For example

```javascript
import {call, cancel, join, take, put} from "redux-saga/effects"

function* saga() {
  yield take(ACTION)              // Blocking: will wait for the action
  yield call(ApiFn, ...args)      // Blocking: will wait for ApiFn (If ApiFn returns a Promise)
  yield call(otherSaga, ...args)  // Blocking: will wait for otherSaga to terminate

  yield put(...)                   // Non-Blocking: will dispatch within internal scheduler

  const task = yield fork(otherSaga, ...args)  // Non-blocking: will not wait for otherSaga
  yield cancel(task)                           // Non-blocking: will resume immediately
  // or
  yield join(task)                              // Blocking: will wait for the task to terminate
}
```

### Watcher/Worker

refers to a way of organizing the control flow using two separate Sagas

指使用两个独立 Sagas 来组织控制流的一种方式（即一个用于接收 action，一个用于执行 action，前者扮演 watcher 角色，后者扮演 workder 角色）。

- The watcher: will watch for dispatched actions and fork a worker on every action
- watcher：监听 actions，每当收到匹配的 action 时就创建一个 workder 去执行。

- The worker: will handle the action and terminate
- workder：处理 action 并终止

example

```javascript
function* watcher() {
  while (true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}

function* worker(payload) {
  // ... do some stuff
}
```
