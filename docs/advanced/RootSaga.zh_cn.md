# Setting up your root Saga

In the beginner tutorial it is shown that your root saga will look something like this

在之前的教程中展示的 root saga 看起来是下面这个样子

```javascript
export default function* rootSaga() {
  yield all([
    helloSaga(),
    watchIncrementAsync()
  ])
  // code after all-effect
}
```

This is one of a few ways to implement your root. Here, the `all` effect is used with an array and your sagas will be executed in parallel. Other root implementations may help you better handle errors and more complex data flow.

这是实现 root saga 的方式之一。此例中，向 `all` effect 中传递 saga 数组，数组中的 saga 会并发执行。其它的 root saga 实现方式可能会帮助你更好的处理 error 和构建更复杂的数据流。

Contributor @slorber mentioned in [issue#760](https://github.com/redux-saga/redux-saga/issues/760) several other common root implementations. To start, there is one popular implementation that behaves similarly to the tutorial root saga behavior:

@slorber 在 [issue#760](https://github.com/redux-saga/redux-saga/issues/760) 中提到一些其他常用 root 实现。我们以一个其行为表现和教程中 root saga 示例类似的实现来作为开始：

```javascript
export default function* root() {
  yield fork(saga1)
  yield fork(saga2)
  yield fork(saga3)
  // code after fork-effect
}
```

Using three unique `yield fork` will give back a task descriptor three times. The resulting behavior in your app is that all of your sub-sagas are started and executed in the same order. `fork` is non-blocking and so allows the `rootSaga` in these two cases to finish while the child sagas are kept running and blocked by their internal effects.

使用三个独立的 `yield fork` 会返回 3 次任务描述符（task descriptor）。这样做的结果是你应用中的所有子 saga 会以相同的方式启动和执行。因此在这两种情况下由于 `fork` 是非阻塞调用，它允许 `rootSaga` 在子 saga 继续执行并被它们自身的 effect 阻塞的情况下结束。

The difference between one big all effect and several fork effects are that all effect is blocking, so *code after all-effect* (see comments in above code) is executed after all the children sagas completes, while fork effects are non-blocking and *code after fork-effect* gets executed right after yielding fork effects. Another difference is that you can get task descriptors when using fork effects, so in the subsequent code you can cancel/join the forked task via task descriptors.

使用 all effect 来包含所有子 saga 和使用多个 fork effect 这两种方式最大的不同就是 all effect 是阻塞的，因此 *all-effect 之后的代码* （看上面代码中的注释）会在所有子 saga 完成之后才会执行，然而 fork effect 是非阻塞的因此 *fork-effect 之后的代码* 会紧接着 fork effect 执行。另一个不同就是当你使用 fork effect 的时候你可以获得任务描述符，因此在接下来的代码中你可以通过任务描述符对 fork 出来的任务使用 cancel/join。

## Nesting fork effects in all effect

```javascript
const [task1, task2, task3] = yield all([ fork(saga1), fork(saga2), fork(saga3) ])
```

There is another popular pattern when designing root saga: nesting `fork` effects in an `all` effect. By doing so, you can get an array of task descriptors, and the code after the `all` effect will be executed immediately because each `fork` effect is non-blocking and synchronously returning a task descriptor.

在设计 root saga 的时候还有另一种流行的模式：在 `all` 中嵌套 `fork`。这样做之后，你能够得到一个任务描述符数组，并且由于 `fork` 是非阻塞的并且会同步返回一个任务描述符，因此 `all` effect 之后的代码能够立即执行。

Note that though `fork` effects are nested in an `all` effect, they are always connected to the parent task through the underlying forkQueue. Uncaught errors from forked tasks bubble to the parent task and thus abort it (and all its child tasks) - they cannot be caught by the parent task.

注意，尽管 `fork` 被嵌套在了 `all` 中，它们仍然基于 fork 队列（forkQueue）与父级保持联系。fork 任务中未捕获的 error 会冒泡到父级任务中，并且会导致它退出（包括所有的子任务） —— 它们不能被父任务捕获。

## Avoid nesting fork effects in race effect

```javascript
// DO NOT DO THIS. The fork effect always win the race immediately.
yield race([
  fork(someSaga),
  take('SOME-ACTION'),
  somePromise,
])
```

On the other hand, `fork` effects in a `race` effect is most likely a bug. In the above code, since `fork` effects are non-blocking, they will always win the race immediately.

另一方面，`race` effect 中放 `fork` effect 简直就是 bug。在上面的代码中，由于 `fork` effect 是非阻塞的，因此它们始终会在 race 中立即胜出（立即返回）。

## Keeping the root alive

In practice, these implementations aren't terribly practical because your rootSaga will terminate on the first error in any individual child effect or saga and crash your whole app! Ajax requests in particular would put your app at the mercy of the status of any endpoints your app makes requests to.

在实际应用中，上面那些实现方式实际上并不太实用，因为你的 root saga 在遇到第一个来自任意子 effect 或者子 saga 抛出的 error 时会终止并让整个应用崩溃！尤其是 Ajax 请求，它会让你的应用依赖于所有服务端返回的状态。

`spawn` is an effect that will *disconnect* your child saga from its parent, allowing it to fail without crashing it's parent. Obviously, this does not relieve us from our responsibility as developers from still handling errors as they arise. In fact, it's possible that this might obscure certain failures from developer's eyes and cause problems further down the road.

`spawn` effect 可以让你的子 saga 和父级 *断开联系* ，这在子 saga 执行失败的情况下不会导致父级崩溃。显然，这并不是说发生错误时开发人员可以忽略它们。实际上，这可能会蒙蔽开发人员的双眼，从而导致更进一步的问题产生。

The `spawn` effect might be considered similar to [Error Boundaries](https://reactjs.org/docs/error-boundaries.html) in React in that it can be used as extra safety measure at some level of the saga tree, cutting off a single feature or something and not letting the whole app crash. The difference is that there is no special syntax like the `componentDidCatch` that exists for React Error Boundaries. You must still write your own error handling and recovery code.

可以把 `spawn` effect 考虑成 React 中的 [Error Boundaries](https://reactjs.org/docs/error-boundaries.html)，它可以当做是 saga 树中某个层级的额外安全手段，用于拦截单个特性或一些其他问题，从而不至于让整个应用崩溃。但不同的是它并没有像 React 错误边界中 `componentDidCatch` 这样特定的语法。你仍然需要自己编写错误处理和错误恢复代码。

```javascript
export default function* root() {
  yield spawn(saga1)
  yield spawn(saga2)
  yield spawn(saga3)
}
```

## Keeping everything alive

In some cases, it may be desirable for your sagas to be able to restart in the event of failure. The benefit is that your app and sagas may continue to work after failing, i.e. a saga that `yield takeEvery(myActionType)`. But we do not recommend this as a blanket solution to keep all sagas alive. It is very likely that it makes more sense to let your saga fail in sanely and predictably and handle/log your error.

在一些情况下，你可能希望错误发生过后能重启 saga。这样做的好处是你的应用和 saga 可以在发生失败之后继续工作。即，一个类似 `yield takeEvery(myActionType)` 的 saga。但我们并不建议用这种方式来保持所有 saga 在线。更好的做法是让你的 saga 即使失败也是以一种可控的方式并可以对这些 error 进行处理或者记录。

For example, @ajwhite offered this scenario as a case where keeping your saga alive would cause more problems than it solves:

例如，@ajwhite 提供了这种场景，即（产生问题之后）保持 saga 继续存活带来的问题可能会比它们解决的问题更多：

```javascript
function* sagaThatMayCrash () {
  // wait for something that happens _during app startup_
  yield take(APP_INITIALIZED)

  // assume it dies here
  yield call(doSomethingThatMayCrash)
}
```
> If the sagaThatMayCrash is restarted, it will restart and wait for an action that only happens once when the application starts up. In this scenario, it restarts, but it never recovers.
>
> 如果 sagaThatMayCrash 重启了，它会重启并等待一个只会在应用启动时触发的 action。在这种场景下，即便是它重启了也永远无法恢复。

But for the specific situations that would benefit from starting, user @granmoe proposed an implementation like this in issue #570:

但是对于某些需要重启的特定场景，用户 @granmoe 在 issue #570 中提出了类似如下的实现：

```javascript
function* rootSaga () {
  const sagas = [
    saga1,
    saga2,
    saga3,
  ];

  yield sagas.map(saga =>
    spawn(function* () {
      while (true) {
        try {
          yield call(saga)
          break
        } catch (e) {
          console.log(e)
        }
      }
    })
  );
}
```

This strategy maps our child sagas to spawned generators (detaching them from the root parent) which start our sagas as subtasks in a `try` block. Our saga will run until termination, and then be automatically restarted. The `catch` block harmlessly handles any error that may have been thrown by, and terminated, our saga.

这种实现策略通过将子 saga 映射到每个置于 spawn 的 generator 函数中（断开与父级的联系）来将每个 saga 当做一个子 saga 运行在 `try` 语句块中。我们的 saga 会一直运行直到结束，然后自动重启。`catch` 块可以处理抛出的任何可能会导致 saga 终止的错误。