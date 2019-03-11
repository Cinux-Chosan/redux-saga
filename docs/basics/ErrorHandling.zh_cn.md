# Error handling

In this section we'll see how to handle the failure case from the previous example. Let's suppose that our API function `Api.fetch` returns a Promise which gets rejected when the remote fetch fails for some reason.

本章节我们探讨如何对前面的示例做错误处理。假设出于某些未知原因 API `Api.fetch` 返回了一个 rejected Promise。

We want to handle those errors inside our Saga by dispatching a `PRODUCTS_REQUEST_FAILED` action to the Store.

这种情况我们希望通过在 Saga 内部 dispatch 一个 `PRODUCTS_REQUEST_FAILED` action 到 Store。

We can catch errors inside the Saga using the familiar `try/catch` syntax.

我们可以通过在 Saga 内部使用熟悉的 `try/catch` 语法来捕获错误。

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

// ...

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}
```

In order to test the failure case, we'll use the `throw` method of the Generator

为了测试失败的情况，我们需要用到 Generator 的 `throw` 方法

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

// create a fake error
const error = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_REQUEST_FAILED', error })"
)
```

In this case, we're passing the `throw` method a fake error. This will cause the Generator to break the current flow and execute the catch block.

现在的情况是，我们通过 `throw` 方法传递了一个错误消息到 Generator。这会导致 Generator 跳出当前执行过程转而执行 catch 代码块。

Of course, you're not forced to handle your API errors inside `try`/`catch` blocks. You can also make your API service return a normal value with some error flag on it. For example, you can catch Promise rejections and map them to an object with an error field.

当然，并没有强制你在 `try/catch` 块中处理 API 的错误。你也可以让 API 返回一个包含错误标志的值。例如，你可以捕获 Promise 的 rejections 并将它映射到返回对象的 error 字段。

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

function fetchProductsApi() {
  return Api.fetch('/products')
    .then(response => ({ response }))
    .catch(error => ({ error }))
}

function* fetchProducts() {
  const { response, error } = yield call(fetchProductsApi)
  if (response)
    yield put({ type: 'PRODUCTS_RECEIVED', products: response })
  else
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
}
```

## onError hook
Errors in forked tasks [bubble up to their parents](../api/README.md#error-propagation)
until it is caught or reaches the root saga.
If an error propagates to the root saga the whole saga tree is already **terminated**. The preferred approach, in this case, to use [onError hook](../api/README.md#error-propagation#createsagamiddlewareoptions) to report an exception, inform a user about the problem and gracefully terminate your app.

这些任务分叉中出现的错误会[冒泡到父级](../api/README.md#error-propagation) 直到被捕获或到达根 saga。如果一个错误冒泡到根 saga 则整个 saga 树**已经终止**。这种情况下首选方式就是使用 [onError hook](../api/README.md#error-propagation#createsagamiddlewareoptions) 来报告异常，告诉用户发生了什么问题然后优雅的终止你的应用。

Why I cannot use `onError` hook as a global error handler?
Usually, there is no one-size-fits-all solution, as exceptions are context dependent. Consider `onError` hook as the last resort that helps you to handle **unexpected** errors.

为什么我不把 `onError` 用作全局错误处理函数？通常，由于异常一般都依赖于执行上下文，因此没有一种可以满足所有情况的解决方案。请将 `onError` 当做处理未知错误的最后手段。 

What if I don't want an error to bubble?
Consider to use safe wrapper. You can find examples [here](https://github.com/redux-saga/redux-saga/issues/1250)

如果不希望错误冒泡怎么办？请考虑使用安全的封装器。你可以在[这里](https://github.com/redux-saga/redux-saga/issues/1250)找到示例。