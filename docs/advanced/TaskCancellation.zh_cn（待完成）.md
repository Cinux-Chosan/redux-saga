# Task cancellation

We saw already an example of cancellation in the [Non blocking calls](NonBlockingCalls.md) section. In this section we'll review cancellation in more detail.

我们已经在之前的 [非阻塞调用](NonBlockingCalls.md) 部分看到过关于取消任务的例子了。现在我们来更详细的回顾一遍。

Once a task is forked, you can abort its execution using `yield cancel(task)`.

一旦一个任务被创建，你可以使用 `yield cancel(task)` 来取消对它的继续执行。

To see how it works, let's consider a basic example: A background sync which can be started/stopped by some UI commands. Upon receiving a `START_BACKGROUND_SYNC` action, we fork a background task that will periodically sync some data from a remote server.

我们通过一个基础的示例来说明它是如何工作的：用户界面可以 开始/停止 后台数据同步。在接收到 `START_BACKGROUND_SYNC` action 后，我们将派生一个后台任务，该任务将定期从远程服务器同步某些数据。

The task will execute continually until a `STOP_BACKGROUND_SYNC` action is triggered. Then we cancel the background task and wait again for the next `START_BACKGROUND_SYNC` action.

该任务会一直执行直到有 `STOP_BACKGROUND_SYNC` action 被触发。然后我们取消后台任务并重新等待下一个 `START_BACKGROUND_SYNC` action。

```javascript
import { take, put, call, fork, cancel, cancelled, delay } from 'redux-saga/effects'
import { someApi, actions } from 'somewhere'

function* bgSync() {
  try {
    while (true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield delay(5000)
    }
  } finally {
    if (yield cancelled())
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while ( yield take(START_BACKGROUND_SYNC) ) {
    // starts the task in the background
    const bgSyncTask = yield fork(bgSync)

    // wait for the user stop action
    yield take(STOP_BACKGROUND_SYNC)
    // user clicked stop. cancel the background task
    // this will cause the forked bgSync task to jump into its finally block
    yield cancel(bgSyncTask)
  }
}
```

In the above example, cancellation of `bgSyncTask` will use [Generator.prototype.return](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/return) to make the Generator jump directly to the finally block. Here you can use `yield cancelled()` to check if the Generator has been cancelled or not.

上面的例子中，通过 [Generator.prototype.return](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/return) 使 Generator 直接跳到 finally 块来取消 `bgSyncTask`。这里我们可以使用 `yield cancelled()` 来检测 Generator 是否已经被取消了。

Cancelling a running task will also cancel the current Effect where the task is blocked at the moment of cancellation.

取消正在运行的任务将同时会取消此刻任务被阻塞的 Effect。

For example, suppose that at a certain point in an application's lifetime, we have this pending call chain:

例如，假设在应用生命周期的某个时刻，我们有一连串处于挂起状态的调用：

```javascript
function* main() {
  const task = yield fork(subtask)
  ...
  // later
  yield cancel(task)
}

function* subtask() {
  ...
  yield call(subtask2) // currently blocked on this call
  ...
}

function* subtask2() {
  ...
  yield call(someApi) // currently blocked on this call
  ...
}
```

`yield cancel(task)` triggers a cancellation on `subtask`, which in turn triggers a cancellation on `subtask2`.

`yield cancel(task)` 触发对 `subtask` 的取消操作，这也会导致 `subtask2` 被取消。

So we saw that Cancellation propagates downward (in contrast returned values and uncaught errors propagates upward). You can see it as a *contract* between the caller (which invokes the async operation) and the callee (the invoked operation). The callee is responsible for performing the operation. If it has completed (either success or error) the outcome propagates up to its caller and eventually to the caller of the caller and so on. That is, callees are responsible for *completing the flow*.

因此我们看到取消会向下传播（相反，返回值和未捕获异常会向上传播）。您可以将其视为调用方（调用异步操作的一方）和被调用方（被调用操作）之间的约定。被调用方负责执行操作。如果它已经执行完成（不管成功或失败），那结果会向上传播给调用者甚至调用者的调用者等等。也就是说，被调用方负责 *完成该流程* 。

Now if the callee is still pending and the caller decides to cancel the operation, it triggers a kind of a signal that propagates down to the callee (and possibly to any deep operations called by the callee itself). All deeply pending operations will be cancelled.

如果现在被调用方仍然处于挂起状态并且调用方决定取消该操作，它会向下传播一种信号给被调用方（也可能是被调用者调用的任意深层级的操作）。所有更深层次的挂起操作都将被取消。

There is another direction where the cancellation propagates to as well: the joiners of a task (those blocked on a `yield join(task)`) will also be cancelled if the joined task is cancelled. Similarly, any potential callers of those joiners will be cancelled as well (because they are blocked on an operation that has been cancelled from outside).

## Testing generators with fork effect

When `fork` is called it starts the task in the background and also returns task object like we have learned previously. When testing this we have to use utility function `createMockTask`. Object returned from this function should be passed to next `next` call after fork test. Mock task can then be passed to `cancel` for example. Here is test for `main` generator which is on top of this page.

当 `fork` 被调用时，它会启动一个后台任务同时也会返回一个我们之前见到的那种任务对象。当需要对它进行测试时，我们需要用到工具函数 `createMockTask`。该函数返回的对象需要传递给 fork 后的下一个 `next` 调用。Mock 任务会被传递给 `cancel`。这里演示了如何对本页面顶部的 `main` generator 进行测试。

```javascript
import { createMockTask } from '@redux-saga/testing-utils';

describe('main', () => {
  const generator = main();

  it('waits for start action', () => {
    const expectedYield = take(START_BACKGROUND_SYNC);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('forks the service', () => {
    const expectedYield = fork(bgSync);
    const mockedAction = { type: 'START_BACKGROUND_SYNC' };
    expect(generator.next(mockedAction).value).to.deep.equal(expectedYield);
  });

  it('waits for stop action and then cancels the service', () => {
    const mockTask = createMockTask();

    const expectedTakeYield = take(STOP_BACKGROUND_SYNC);
    expect(generator.next(mockTask).value).to.deep.equal(expectedTakeYield);

    const expectedCancelYield = cancel(mockTask);
    expect(generator.next().value).to.deep.equal(expectedCancelYield);
  });
});
```

You can also use mock task's functions `setRunning`, `setResult` and `setError` to set mock task's state. For example `mockTask.setRunning(false)`.

### Note

It's important to remember that `yield cancel(task)` doesn't wait for the cancelled task to finish (i.e. to perform its finally block). The cancel effect behaves like fork. It returns as soon as the cancel was initiated. Once cancelled, a task should normally return as soon as it finishes its cleanup logic.

注意，请记住 `yield cancel(task)` 不会等待被取消的任务完成（即，执行它的 finally 语句块）。 `cancel` 的表现类似于 `fork`。启用 cancel 之后它会立即返回。一旦取消完成，任务应该在完成它的清理逻辑之后正常返回。

## Automatic cancellation

Besides manual cancellation there are cases where cancellation is triggered automatically

除了手动取消，以下场景会自动触发取消操作：

1. In a `race` effect. All race competitors, except the winner, are automatically cancelled.

1. 在 `race` effect 中。除了最先完成的任务，其它的都会自动被取消。

2. In a parallel effect (`yield all([...])`). The parallel effect is rejected as soon as one of the sub-effects is rejected (as implied by `Promise.all`). In this case, all the other sub-effects are automatically cancelled.

2. 在并行 effect （`yield all([...])`）中。一旦其中某个子 effect 被拒绝时，并行 effect 也会被拒绝（如同 `Promise.all`）。在这种情况下，所有其它子 effect 会被自动取消。