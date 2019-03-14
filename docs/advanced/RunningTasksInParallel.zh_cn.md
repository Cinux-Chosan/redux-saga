# Running Tasks In Parallel

The `yield` statement is great for representing asynchronous control flow in a linear style, but we also need to do things in parallel. We can't write:

`yield` 非常适合用线性方式来表示异步控制流，但我们仍然需要做一些并行的事情。我们不能这样写：

```javascript
// wrong, effects will be executed in sequence
const users = yield call(fetch, '/users')
const repos = yield call(fetch, '/repos')
```

Because the 2nd effect will not get executed until the first call resolves. Instead we have to write:

因为第二个 effect 会一直等到第一个完成后才会执行。因此我们需要像下面这样写：

```javascript
import { all, call } from 'redux-saga/effects'

// correct, effects will get executed in parallel
const [users, repos] = yield all([
  call(fetch, '/users'),
  call(fetch, '/repos')
])
```

When we yield an array of effects, the generator is blocked until all the effects are resolved or as soon as one is rejected (just like how `Promise.all` behaves).

当我们 yield 一个 effects 数组时，generator 会一直被阻塞直到所有的 effects 完成或者某一个先变为 rejected（和 `Promise.all` 行为差不多）。