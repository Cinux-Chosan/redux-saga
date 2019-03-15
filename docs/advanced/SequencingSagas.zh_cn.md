# Sequencing Sagas via `yield*`

You can use the builtin `yield*` operator to compose multiple Sagas in a sequential way. This allows you to sequence your *macro-tasks* in a procedural style.

你可以使用内置的 `yield*` 操作符将多个 Sagas 按顺序组合在一起。这使得你能够以传统的过程化风格对 *宏任务* 进行排序。

```javascript
function* playLevelOne() { ... }

function* playLevelTwo() { ... }

function* playLevelThree() { ... }

function* game() {
  const score1 = yield* playLevelOne()
  yield put(showScore(score1))

  const score2 = yield* playLevelTwo()
  yield put(showScore(score2))

  const score3 = yield* playLevelThree()
  yield put(showScore(score3))
}
```

Note that using `yield*` will cause the JavaScript runtime to *spread* the whole sequence. The resulting iterator (from `game()`) will yield all values from the nested iterators. A more powerful alternative is to use the more generic middleware composition mechanism.

注意，使用 `yield*` 会导致 JavaScript 运行时 *展开* 整个序列，最终的迭代器（来自`game()`）会 yield 所有嵌套迭代器的值（译者注：这是生成器的特性，即相当于把嵌套在它内部的生成器的值会和自己的值在同一层级返回，相当于平铺了所有的值）。一种更好的替代方案是使用通用的中间件组合机制。