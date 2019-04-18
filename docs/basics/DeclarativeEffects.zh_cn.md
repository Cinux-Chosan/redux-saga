# Declarative Effects

In `redux-saga`, Sagas are implemented using Generator functions. To express the Saga logic, we yield plain JavaScript Objects from the Generator. We call those Objects [*Effects*](https://redux-saga.js.org/docs/api/#effect-creators). An Effect is an object that contains some information to be interpreted by the middleware. You can view Effects like instructions to the middleware to perform some operation (e.g., invoke some asynchronous function, dispatch an action to the store, etc.).

在 `redux-saga` 中， Sagas 是通过 Generator 函数来实现的。为了表达 Saga 的逻辑，我们从 Generator 中 yield 出纯 JavaScript 对象。我们将这些对象称作 [*Effects*](https://redux-saga.js.org/docs/api/#effect-creators)。Effect 是一个包含了中间件需要识别和处理信息的对象。你可以把 Effect 看做是引领中间件去完成指定操作的引导人（如引导它调用异步函数、向 store 发送 action 等。）。

To create Effects, you use the functions provided by the library in the `redux-saga/effects` package.

使用 `redux-saga/effects` 库提供的函数来创建 Effect。

In this section and the following, we will introduce some basic Effects. And see how the concept allows the Sagas to be easily tested.

在接下来的部分，我们将介绍一些基本的 Effect。并且你可以从中了解到这种概念（指 Effects）是如何使得 Saga 易于测试的。

Sagas can yield Effects in multiple forms. The easiest way is to yield a Promise.

Saga 能够以多种形式抛出（yield）Effect。最简单的方式就是抛出（yield）一个 Promise。

For example suppose we have a Saga that watches a `PRODUCTS_REQUESTED` action. On each matching action, it starts a task to fetch a list of products from a server.

假设我们有个监听 `PRODUCTS_REQUESTED` action 的 Saga。（我们需要它）在每接收到一个匹配的 action 时，就启动一个任务去从服务端获取产品列表数据。

```javascript
import { takeEvery } from 'redux-saga/effects'
import Api from './path/to/api'

function* watchFetchProducts() {
  yield takeEvery('PRODUCTS_REQUESTED', fetchProducts)
}

function* fetchProducts() {
  const products = yield Api.fetch('/products')
  console.log(products)
}
```

In the example above, we are invoking `Api.fetch` directly from inside the Generator (In Generator functions, any expression at the right of `yield` is evaluated then the result is yielded to the caller).

上面的例子中，我们直接在 Generator 内部调用了 `Api.fetch`（在 Generator 函数中，任何在 `yield` 右边的表达式都会在返回给调用者前被执行并把执行结果返回给 Generator 的调用者）。

`Api.fetch('/products')` triggers an AJAX request and returns a Promise that will resolve with the resolved response, the AJAX request will be executed immediately. Simple and idiomatic, but...

`Api.fetch('/products')` 发出一个 AJAX 请求并返回一个 Promise，AJAX 请求会被立即执行。这种做法既简单又符合我们的思维惯性，但是...

Suppose we want to test the generator above:

假如我们希望对上面的 Generator 进行测试：

```javascript
const iterator = fetchProducts()
assert.deepEqual(iterator.next().value, ??) // what do we expect ?
```

We want to check the result of the first value yielded by the generator. In our case it's the result of running `Api.fetch('/products')` which is a Promise . Executing the real service during tests is neither a viable nor practical approach, so we have to *mock* the `Api.fetch` function, i.e. we'll have to replace the real function with a fake one which doesn't actually run the AJAX request but only checks that we've called `Api.fetch` with the right arguments (`'/products'` in our case).

我们希望对生成器抛出（yield）的第一个值进行检查。在我们目前的情况下，它是执行 `Api.fetch('/products')` 后返回的一个 Promise。在测试期间去执行真正的 service 既不可行也不切实际，因此我们会对 `Api.fetch` 函数进行 *模拟* ，即用一个假的 `Api.fetch` 来代替真实的 `Api.fetch`，这个假的 `Api.fetch` 不会去真正的发送 AJAX 请求，它需要做的就只是检查我们是否使用了正确的参数（这里的参数就是 `'/products'`）来调用 `Api.fetch`。

Mocks make testing more difficult and less reliable. On the other hand, functions that return values are easier to test, since we can use a simple `equal()` to check the result. This is the way to write the most reliable tests.

这样模拟使得测试变得困难又不可靠。另一方面，直接返回值的函数更易于测试，因为我们可以使用 `equal()` 对返回结果进行等价测试。这也是编写大多数高可靠测试代码的一种方式。

Not convinced? I encourage you to read [Eric Elliott's article](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d#.4ttnnzpgc):

不相信吗？我建议你阅读 [Eric Elliott's article](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d#.4ttnnzpgc)：

> (...)`equal()`, by nature answers the two most important questions every unit test must answer,
but most don’t:
>
> 本质上每一个单元测试中 `equal()` 都必须回答两个很重要但大多测试都没有回答的问题：

- What is the actual output?
- 实际的输出是什么
- What is the expected output?
- 期望的输出是什么
>
> If you finish a test without answering those two questions, you don’t have a real unit test. You have a sloppy, half-baked test.
>
> 如果你没有回答这两个问题就完成了测试，那你这个就不是真正的单元测试，而是一个草率的半生不熟的测试。

What we actually need to do is make sure the `fetchProducts` task yields a call with the right function and the right arguments.

实际上我们需要做的就是确保 `fetchProducts` 任务能抛出（yield）一个带有正确的函数和参数的 call（一个 redux-saga 提供的函数）。

Instead of invoking the asynchronous function directly from inside the Generator, **we can yield only a description of the function invocation**. i.e. We'll yield an object which looks like

与其在 Generator 内部直接调用异步函数，**我们更推荐只 yield 一个函数调用的描述信息**。即 yield 一个类似如下结构的对象：

```javascript
// Effect -> call the function Api.fetch with `./products` as argument
{
  CALL: {
    fn: Api.fetch,
    args: ['./products']
  }
}
```

Put another way, the Generator will yield plain Objects containing *instructions*, and the `redux-saga` middleware will take care of executing those instructions and giving back the result of their execution to the Generator. This way, when testing the Generator, all we need to do is to check that it yields the expected instruction by doing a simple `deepEqual` on the yielded Object.

换句话说，Generator 抛出（yield）一个包含 *指令* 的纯对象，`redux-saga` 中间件会小心地执行这些指令并把结果返回给 Generator。采用这种方式以后，当测试 Generator 时，我们所要做的就只是对抛出的对象调用 `deepEqual` 来检查它是否是我们预期的指令。

For this reason, the library provides a different way to perform asynchronous calls.

正是出于这样的原因，redux-saga 才提供了一种与众不同的方式来执行异步调用。

```javascript
import { call } from 'redux-saga/effects'

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // ...
}
```

We're using now the `call(fn, ...args)` function. **The difference from the preceding example is that now we're not executing the fetch call immediately, instead, `call` creates a description of the effect**. Just as in Redux you use action creators to create a plain object describing the action that will get executed by the Store, `call` creates a plain object describing the function call. The redux-saga middleware takes care of executing the function call and resuming the generator with the resolved response.

我们现在使用 `call(fn, ...args)` 函数。 **与前面的示例不同的是现在我们并没有立即执行 fetch 调用，而是用 `call` 来创建了一个 Effect 的描述。** 就跟在 Redux 中用 action 创建器创建一个会被 Store 执行的用于描述 action 的纯对象一样，`call` 创建了一个纯对象用于描述需要执行的函数调用。redux-saga 中间件会小心地执行该函数调用并用执行完成后的结果来恢复执行 Generator。

This allows us to easily test the Generator outside the Redux environment. Because `call` is just a function which returns a plain Object.

由于 `call` 就只是一个返回纯对象的函数。这使得我们可以在 Redux 环境之外也能够很轻松对 Generator 进行测试。

```javascript
import { call } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)
```

Now we don't need to mock anything, and a basic equality test will suffice.

现在我们不需要模拟任何东西，只需要做一些基本的等价测试就够了。

The advantage of those *declarative calls* is that we can test all the logic inside a Saga by iterating over the Generator and doing a `deepEqual` test on the values yielded successively. This is a real benefit, as your complex asynchronous operations are no longer black boxes, and you can test in detail their operational logic no matter how complex it is.

**声明性调用** 的好处是我们可以对 Generator 进行迭代并用 `deepEqual` 来对每个 Generator 抛出的值进行等价测试的这种方式来对 Saga 中所有的逻辑进行测试。这样做会带来很多好处，因为那些复杂的异步调用操作对你来说不再是黑盒，不管它的操作逻辑有多复杂你都可以对它进行详尽的测试。

`call` also supports invoking object methods, you can provide a `this` context to the invoked functions using the following form:

`call` 也支持调用对象方法，你可以向下面这样给它提供一个 `this` 作为上下文：

```javascript
yield call([obj, obj.method], arg1, arg2, ...) // as if we did obj.method(arg1, arg2 ...)
```

`apply` is an alias for the method invocation form

`apply` 是该方法的另一种调用形式

```javascript
yield apply(obj, obj.method, [arg1, arg2, ...])
```

`call` and `apply` are well suited for functions that return Promise results. Another function `cps` can be used to handle Node style functions (e.g. `fn(...args, callback)` where `callback` is of the form `(error, result) => ()`). `cps` stands for Continuation Passing Style.

`call` 和 `apply` 都非常适合返回 Promise 的函数调用。另一个函数 `cps` 可以用于处理 Node 形式的函数 (如 `fn(...args, callback)`，其中 `callback` 是 `(error, result) => ()` 这种格式）。`cps` 代表 **C**ontinuation **P**assing **S**tyle。

For example:

```javascript
import { cps } from 'redux-saga/effects'

const content = yield cps(readFile, '/path/to/file')
```

And of course you can test it just like you test `call`:

当然你也可以像测试 `call` 那样测试它：

```javascript
import { cps } from 'redux-saga/effects'

const iterator = fetchSaga()
assert.deepEqual(iterator.next().value, cps(readFile, '/path/to/file') )
```

`cps` also supports the same method invocation form as `call`.

`cps` 也支持 `call` 一样的调用格式（即指定 `this` 作为函数执行上下文）。

A full list of declarative effects can be found in the [API reference](https://redux-saga.js.org/docs/api/#effect-creators).

关于声明式 effect 完整的信息可以在 [API 参考](https://redux-saga.js.org/docs/api/#effect-creators) 中找到。