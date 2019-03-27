# redux-saga's fork model

In `redux-saga` you can dynamically fork tasks that execute in the background using 2 Effects

在 `redux-saga` 中，你可以用下面两个 Effects 创建后台执行的任务。

- `fork` is used to create *attached forks*
- `fork` 用于创建 *关联任务分支*
- `spawn` is used to create *detached forks*
- `spawn` 用于创建 *独立任务分支*

## Attached forks (using `fork`)

Attached forks remain attached to their parent by the following rules

关联任务分支通过下面的规则保留了其与父级的关联

### Completion

- A Saga terminates only after
- Saga 只会在下面的情况完成后结束
  - It terminates its own body of instructions
  - 它自身的指令结束
  - All attached forks are themselves terminated
  - 所有关联的任务分支执行结束

For example say we have the following

举个例子

```js
import { fork, call, put, delay } from 'redux-saga/effects'
import api from './somewhere/api' // app specific
import { receiveData } from './somewhere/actions' // app specific

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield delay(1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  yield call(fetchAll)
}
```

`call(fetchAll)` will terminate after:

`call(fetchAll)` 会在满足下面条件之后结束：

- The `fetchAll` body itself terminates, this means all 3 effects are performed. Since `fork` effects are non blocking, the
task will block on `delay(1000)`

- `fetchAll` 自身结束，意思就是所有的 3 个 effects 都已经被执行，由于 `fork` effect 是非阻塞的，因此该任务会在 `delay(1000)` 处阻塞。

- The 2 forked tasks terminate, i.e. after fetching the required resources and putting the corresponding `receiveData` actions

- 两个 fork 任务都执行结束，即获取到需要的资源并执行相应的 `receiveData` action

So the whole task will block until a delay of 1000 millisecond passed *and* both `task1` and `task2` finished their business.

因此整个任务会阻塞直到 1000 毫秒之后 *并且* `task1` 和 `task2` 都执行完成了它们的任务。

Say for example, the delay of 1000 milliseconds elapsed and the 2 tasks haven't yet finished, then `fetchAll` will still wait
for all forked tasks to finish before terminating the whole task.

比如说，1000 毫秒延迟之后并且 2 个任务没有结束，则 `fetchAll` 会等待所有的 fork 任务结束之后再结束整个任务。

The attentive reader might have noticed the `fetchAll` saga could be rewritten using the parallel Effect

细心的读者也许已经发现可以通过并发 Effect 来重写 `fetchAll`

```js
function* fetchAll() {
  yield all([
    call(fetchResource, 'users'),     // task1
    call(fetchResource, 'comments'),  // task2,
    delay(1000)
  ])
}
```

In fact, attached forks shares the same semantics with the parallel Effect:

实际上，关联任务分支拥有与并发 Effect 相同的语义：

- We're executing tasks in parallel
- 所有任务并发执行
- The parent will terminate after all launched tasks terminate
- 所有启动的任务结束之后父任务才会结束

And this applies for all other semantics as well (error and cancellation propagation). You can understand how
attached forks behave by considering it as a *dynamic parallel* Effect.

这也适用于所有其它语义（error 和 cancel 的传播）。你可以将关联任务分支理解为 *动态并发* Effect。

## Error propagation

Following the same analogy, Let's examine in detail how errors are handled in parallel Effects

遵循相同的规则，我们来详细查看一下如何处理并发 Effect 中的错误。

for example, let's say we have this Effect

假如我们有下面的 Effect

```js
yield all([
  call(fetchResource, 'users'),
  call(fetchResource, 'comments'),
  delay(1000)
])
```

The above effect will fail as soon as any one of the 3 child Effects fails. Furthermore, the uncaught error will cause
the parallel Effect to cancel all the other pending Effects. So for example if `call(fetchResource, 'users')` raises an
uncaught error, the parallel Effect will cancel the 2 other tasks (if they are still pending) then aborts itself with the
same error from the failed call.

上面的 effect 会在 3 个子 Effect 中任意一个 effect 失败时失败。此外，未捕获的 error 会导致并发 Effect（指 all）将其它挂起的 Effect 取消。因此该例子中如果 `call(fetchResource, 'users')` 抛出一个错误，并发 Effect 会将另外两个任务取消（如果它们仍然处于挂起状态）然后会用相同的错误终止自己。

Similarly for attached forks, a Saga aborts as soon as

Saga 与关联任务分支类似，会在下面的情况下立即终止

- Its main body of instructions throws an error

- 它自身的逻辑指令抛出 error

- An uncaught error was raised by one of its attached forks

- 它内部的关联任务分支抛出未捕获 error

So in the previous example

因此在前面的例子中

```js
//... imports

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield delay(1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  try {
    yield call(fetchAll)
  } catch (e) {
    // handle fetchAll errors
  }
}
```

If at a moment, for example, `fetchAll` is blocked on the `delay(1000)` Effect, and say, `task1` failed, then the whole
`fetchAll` task will fail causing

举个例子，如果在某一时刻 `fetchAll` 阻塞在 `delay(1000)` Effect 的地方，此时 `task1` 执行失败了，整个 `fetchAll` 任务将会失败，导致：

- Cancellation of all other pending tasks. This includes:
- 取消所有其它处于挂起的任务，包括：
  - The *main task* (the body of `fetchAll`): cancelling it means cancelling the current Effect `delay(1000)`
  - *主任务* （`fetchAll` 自身内部）：取消它就意味着取消当前的 Effect `delay(1000)`
  - The other forked tasks which are still pending. i.e. `task2` in our example.
  - 其它处于挂起的关联分支任务。即上例的 `task2`

- The `call(fetchAll)` will raise itself an error which will be caught in the `catch` body of `main`
- `call(fetchAll)` 将会抛出一个 error 到自身，它会被 `main` 内部的 `catch` 捕获。

Note we're able to catch the error from `call(fetchAll)` inside `main` only because we're using a blocking call. And that
we can't catch the error directly from `fetchAll`. This is a rule of thumb, **you can't catch errors from forked tasks**. A failure
in an attached fork will cause the forking parent to abort (Just like there is no way to catch an error *inside* a parallel Effect, only from
outside by blocking on the parallel Effect).

注意，我们能够在 `main` 中捕获从 `call(fetchAll)` 抛出的 error 是因为我们使用了阻塞调用（blocking call）。我们并不能捕获直接来自与 `fetchAll` 的 error。这是经验总结， **你不能捕获来自关联分支任务的 error** 。关联分支任务失败会导致父级终止（就像没有办法捕获并发 Effect 中的 error 一样，只能通过在外部阻塞并发 Effect 来完成）

## Cancellation

Cancelling a Saga causes the cancellation of:

- The *main task* this means cancelling the current Effect where the Saga is blocked

- All attached forks that are still executing


**WIP**

## Detached forks (using `spawn`)

Detached forks live in their own execution context. A parent doesn't wait for detached forks to terminate. Uncaught
errors from spawned tasks are not bubbled up to the parent. And cancelling a parent doesn't automatically cancel detached
forks (you need to cancel them explicitly).

In short, detached forks behave like root Sagas started directly using the `middleware.run` API.


**WIP**
