# Dispatching actions to the store

Taking the previous example further, let's say that after each save, we want to dispatch some action
to notify the Store that the fetch has succeeded (we'll omit the failure case for the moment).

继续前面的例子，我们希望在每次保存成功之后都向 Store 发送一个 action 用于通知它 fetch 操作成功（目前不考虑请求失败的情况）。

We could pass the Store's `dispatch` function to the Generator. Then the
Generator could invoke it after receiving the fetch response:

我们可以把 Store 的 `dispatch` 方法传给 Generator。然后 Generator 在收到响应后去调用它：

```javascript
// ...

function* fetchProducts(dispatch) {
  const products = yield call(Api.fetch, '/products')
  dispatch({ type: 'PRODUCTS_RECEIVED', products })
}
```

However, this solution has the same drawbacks as invoking functions directly from inside the Generator (as discussed in the previous section). If we want to test that `fetchProducts` performs
the dispatch after receiving the AJAX response, we'll need again to mock the `dispatch`
function.

然而，这种方案具有前面章节中提到的在 Generator 内部直接调用函数相同的问题。如果我们想要对 `fetchProducts` 收到 AJAX 响应之后执行的 dispatch 的操作进行测试，我们不得不再次去模拟 `dispatch` 函数。

Instead, we need the same declarative solution. Create a plain JavaScript Object to instruct the
middleware that we need to dispatch some action, and let the middleware perform the real
dispatch. This way we can test the Generator's dispatch in the same way: by inspecting
the yielded Effect and making sure it contains the correct instructions.

取而代之的是，我们可以使用上一章节中提到的声明方案。通过创建一个纯 JavaScript 对象来告诉中间件我们需要 dispatch 一些 action，真正的 dispatch 操作交由中间件来处理。采用这种方式我们就能够以相同的方式来对 Generator 中的 dispatch 进行测试：即对 Generator 抛出的 Effect 进行检查以确保它包含了正确的执行指令。

The library provides, for this purpose, another function `put` which creates the dispatch
Effect.

正是出于这样的目的，`redux-saga/effects` 提供了另一个函数 `put` 来创建 dispatch Effect。

```javascript
import { call, put } from 'redux-saga/effects'
// ...

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // create and yield a dispatch Effect
  yield put({ type: 'PRODUCTS_RECEIVED', products })
}
```

Now, we can test the Generator easily as in the previous section

现在我们又可以像上一章节中提到的那样轻松地对 Generator 进行测试了。

```javascript
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// create a fake response
const products = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.next(products).value,
  put({ type: 'PRODUCTS_RECEIVED', products }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_RECEIVED', products })"
)
```

Note how we pass the fake response to the Generator via its `next` method. Outside the
middleware environment, we have total control over the Generator, we can simulate a
real environment by mocking results and resuming the Generator with them. Mocking
data is a lot easier than mocking functions and spying calls.

注意我们通过 `next` 方法将假的响应数据传递给了 Generator。在中间件之外的环境中我们能够完全控制 Generator，可以用模拟的响应结果来恢复 Generator 从而模拟出一个真实的环境。模拟数据比模拟方法和监视方法的调用要容易得多。