# A common abstraction: Effect

To generalize, triggering Side Effects from inside a Saga is always done by yielding some declarative Effect. (You can also yield Promise directly, but this will make testing difficult as we saw in the first section.)

总的来说，在 Saga 内部要触发一个具有副作用的操作始终应该通过 yield 与之对应的声明性 Effect 来做。（你也可以直接 yield Promise，但这么做会使得测试变得相当困难，就像我们在第一章节中看到的那样。）

What a Saga does is actually compose all those Effects together to implement the desired control flow. The most basic example is to sequence yielded Effects by putting the yields one after another. You can also use the familiar control flow operators (`if`, `while`, `for`) to implement more sophisticated control flows.

Saga 实际上就是将所有这些 Effects 组合在一起来实现我们期望的控制流。最基础的例子就是将这些 Effects 按顺序一个接一个的 yield。你也可以使用一些熟悉的流程控制操作符（`if`, `while`, `for`）来实现更复杂的控制流。

We saw that using Effects like `call` and `put`, combined with high-level APIs like `takeEvery` allows us to achieve the same things as `redux-thunk`, but with the added benefit of easy testability.

使用 `call` 和 `put` 这样的 Effects 再结合 `takeEvery` 这样的高级 API 使得我们能够实现 `redux-thunk` 相同的效果，还具有易于测试的优点。

But `redux-saga` provides another advantage over `redux-thunk`. In the Advanced section you'll encounter some more powerful Effects that let you express complex control flows while still allowing the same testability benefit.

但是 `redux-saga` 还提供了 `redux-thunk` 之外的优势。在高级部分，你将会看到一些更强大的 Effects，它们能够让你实现更复杂的控制流，同时还保持了易于测试的优点。