# Composing Sagas（未校验）

While using `yield*` provides an idiomatic way of composing Sagas, this approach has some limitations:

尽管 `yield*` 提供了一种惯用的 saga 组合方式，但这种方式有一些局限性：

- You'll likely want to test nested generators separately. This leads to some duplication in the test code as well as the overhead of the duplicated execution. We don't want to execute a nested generator but only make sure the call to it was issued with the right argument.
- 当你希望对嵌套的 generators 进行独立测试时。这会导致在测试代码中编写一些重复代码，并且也会带来重复执行的开销。我们并不希望执行嵌套的 generator，只需确保使用了正确的参数来触发对它的调用即可。

- More importantly, `yield*` allows only for sequential composition of tasks, so you can only `yield*` to one generator at a time.
- 更重要的一点，`yield*` 只允许按顺序组合多个任务，因此你同一时间只能对一个 generator 使用 `yield*`。

You can use `yield` to start one or more subtasks in parallel. When yielding a call to a generator, the Saga will wait for the generator to terminate before progressing, then resume with the returned value (or throws if an error propagates from the subtask).

你可以使用 `yield` 来同时启动一个或多个子任务。当 yield 一个对 generator 的 call 时，Saga 将会在继续执行之前等待 generator 运行结束，然后得到返回值并恢复执行（或者当接收到子任务传播过来的错误时抛出异常）。 

```javascript
function* fetchPosts() {
  yield put(actions.requestPosts())
  const products = yield call(fetchApi, '/products')
  yield put(actions.receivePosts(products))
}

function* watchFetch() {
  while (yield take(FETCH_POSTS)) {
    yield call(fetchPosts) // waits for the fetchPosts task to terminate
  }
}
```

Yielding to an array of nested generators will start all the sub-generators in parallel, wait
for them to finish, then resume with all the results

yield 一个嵌套 generator 的数组将会同时启动数组中所有的子 generators，等待它们执行完成，然后得到所有结果并恢复执行。

```javascript
function* mainSaga(getState) {
  const results = yield all([call(task1), call(task2), ...])
  yield put(showResults(results))
}
```

In fact, yielding Sagas is no different than yielding other effects (future actions, timeouts, etc). This means you can combine those Sagas with all the other types using the effect combinators.

实际上，yield Sagas 和 yield 其它 effects 并没有什么不同（future actions， timeouts 等）。这意味着你可以使用 effect 组合器将 Sagas 和其它非 Saga 类型（如下面示例的 delay）组合起来。

For example, you may want the user to finish some game in a limited amount of time:

例如，你希望用户在规定时间内完成游戏：

```javascript
function* game(getState) {
  let finished
  while (!finished) {
    // has to finish in 60 seconds
    const {score, timeout} = yield race({
      score: call(play, getState),
      timeout: delay(60000)
    })

    if (!timeout) {
      finished = true
      yield put(showScore(score))
    }
  }
}
```
