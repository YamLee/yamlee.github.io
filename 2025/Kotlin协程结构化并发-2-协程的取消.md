# Kotlin协程结构化并发-2-协程的取消

> 协程中的取消和异常处理 - 第2部分

*在开发中，就像在生活中一样，我们知道避免做不必要的工作很重要，因为它会浪费内存和能源。这个原则也适用于协程。你需要确保控制协程的生命周期并在不再需要时取消它——这就是结构化并发所代表的内容。继续阅读以了解协程取消的来龙去脉。*

> ⚠️ 为了顺利阅读本文的其余部分，需要先阅读并理解本系列的第1部分。

**系列文章目录：**
- [第1部分：协程基础概念](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)
- [第2部分：协程的取消](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629)
- [第3部分：异常和监督](https://medium.com/androiddevelopers/exceptions-and-supervision-in-coroutines-1cb51b4b24a5)
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

## 调用cancel

当启动多个协程时，跟踪它们或单独取消每个协程可能会很麻烦。相反，我们可以依靠取消协程启动到的整个作用域，因为这将取消所创建的所有子协程：

```kotlin
// 假设我们为应用程序的这一层定义了一个作用域
val job1 = scope.launch { … }
val job2 = scope.launch { … }

scope.cancel()
```

> 取消作用域会取消其子协程

有时你可能只需要取消一个协程，也许是对用户输入的反应。调用`job1.cancel()`确保只有那个特定的协程被取消，所有其他兄弟协程不受影响：

```kotlin
// 假设我们为应用程序的这一层定义了一个作用域
val job1 = scope.launch { … }
val job2 = scope.launch { … }

// 第一个协程将被取消，其他协程不会受到影响
job1.cancel()
```

> 被取消的子协程不会影响其他兄弟协程

协程通过抛出一个特殊的异常来处理取消：`CancellationException`。如果你想提供关于取消原因的更多详细信息，你可以在调用`.cancel`时提供一个`CancellationException`的实例，因为这是完整的方法签名：

```kotlin
fun cancel(cause: CancellationException? = null)
```

如果你不提供自己的`CancellationException`实例，将创建一个默认的`CancellationException`：

```kotlin
public override fun cancel(cause: CancellationException?) {
    cancelInternal(cause ?: defaultCancellationException())
}
```

因为会抛出`CancellationException`，所以你能够使用这个机制来处理协程取消。关于如何做到这一点的更多信息，请参见下面的*处理取消副作用*部分。

在底层，子job通过异常通知其父级关于取消的信息。父级使用取消的原因来确定是否需要处理异常。如果子协程由于`CancellationException`而被取消，则父级无需其他操作。

> ⚠️一旦取消作用域，你就无法在被取消的作用域中启动新协程。

如果你使用androidx KTX库，在大多数情况下你不会创建自己的作用域，因此你不负责取消它们。如果你在`ViewModel`的作用域中工作，使用`viewModelScope`，或者如果你想启动绑定到生命周期作用域的协程，你会使用`lifecycleScope`。`viewModelScope`和`lifecycleScope`都是在合适的时间被取消的`CoroutineScope`对象。例如，当ViewModel被清除时，它会取消在其作用域中启动的协程。

## 为什么我的协程工作没有停止？

如果我们只是调用`cancel`，并不意味着协程工作会停止。如果你正在执行一些相对繁重的计算，比如从多个文件读取，没有什么能自动阻止你的代码运行。

让我们举一个更简单的例子，看看会发生什么。假设我们需要使用协程每秒打印两次"Hello"。我们将让协程运行一秒钟，然后取消它。实现的一个版本可能如下所示：

```kotlin
val startTime = System.currentTimeMillis()
val job = scope.launch {
    var nextPrintTime = startTime
    var i = 0
    while (i < 5) { // 计算密集型循环
        // 每半秒打印一次消息
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("Hello ${i++}")
            nextPrintTime += 500L
        }
    }
}
delay(1000L) // 延迟一秒
println("Cancel!")
job.cancel() // 取消job
println("Done!")
```

让我们逐步看看会发生什么。当调用`launch`时，我们在**活跃**状态下创建了一个新协程。我们让协程运行1000ms。所以现在我们看到打印：

```
Hello 0
Hello 1
Hello 2
```

一旦调用`job.cancel`，我们的协程移动到*取消中*状态。但是，我们看到Hello 3和Hello 4被打印到终端。只有在工作完成后，协程才移动到*已取消*状态。

协程工作不会在调用cancel时立即停止。相反，我们需要修改我们的代码并定期检查协程是否处于活跃状态。

> 协程代码的取消需要是协作式的！

## 让你的协程工作可取消

你需要确保实现的所有协程工作都与取消协作，因此你需要定期检查取消或在开始任何长时间运行的工作之前检查。例如，如果你从磁盘读取多个文件，在开始读取每个文件之前，检查协程是否已被取消。这样你就避免了在不再需要时做CPU密集型工作。

```kotlin
val job = launch {
    for(file in files) {
        // TODO 检查取消
        readFile(file)
    }
}
```

`kotlinx.coroutines`中的所有挂起函数都是可取消的：`withContext`、`delay`等。所以如果你使用任何这些，你不需要检查取消并停止执行或抛出`CancellationException`。但是，如果你没有使用它们，要使你的协程代码协作，我们有两个选项：

- 检查`job.isActive`或`ensureActive()`
- 使用`yield()`让其他工作发生

## 检查job的活跃状态

一个选项是在我们的`while(i<5)`中添加另一个对协程状态的检查：

```kotlin
// 由于我们在launch块中，我们可以访问job.isActive
while (i < 5 && isActive) {
    // 执行工作
    if (System.currentTimeMillis() >= nextPrintTime) {
        println("Hello ${i++}")
        nextPrintTime += 500L
    }
}
```

这意味着我们的工作应该只在协程处于活跃状态时执行。这也意味着一旦我们退出while循环，如果我们想执行一些其他操作，比如记录job是否被取消，我们可以添加对`!isActive`的检查并在那里执行我们的操作。

协程库提供了另一个有用的方法 - `ensureActive()`。它的实现是：

```kotlin
fun Job.ensureActive(): Unit {
    if (!isActive) {
        throw getCancellationException()
    }
}
```

因为这个方法如果job不活跃就会立即抛出异常，我们可以使这成为我们在while循环中做的第一件事：

```kotlin
while (i < 5) {
    ensureActive()
    // 执行工作
    if (System.currentTimeMillis() >= nextPrintTime) {
        println("Hello ${i++}")
        nextPrintTime += 500L
    }
}
```

通过使用`ensureActive`，你避免了实现`isActive`所需的if语句，减少了需要编写的样板代码，但失去了执行任何其他操作（如日志记录）的灵活性。

## 使用yield()让其他工作发生

如果你正在做的工作是1) CPU密集型，2) 可能耗尽线程池，3) 你想允许线程做其他工作而不必向池中添加更多线程，那么使用`yield()`。yield执行的第一个操作将检查完成情况，如果job已经完成，则通过抛出`CancellationException`退出协程。`yield`可以是定期检查中调用的第一个函数，就像上面提到的`ensureActive()`。

```kotlin
val job = scope.launch {
    var nextPrintTime = startTime
    var i = 0
    while (i < 5) {
        yield() // 让出执行权，同时检查取消状态
        // 执行工作
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("Hello ${i++}")
            nextPrintTime += 500L
        }
    }
}
```

## Job.join vs Deferred.await的取消

有两种方法可以等待协程的结果：从`launch`返回的jobs可以调用`join`，从`async`返回的`Deferred`（一种`Job`类型）可以被`await`。

`Job.join`挂起协程直到工作完成。与`job.cancel`一起，它的行为符合你的期望：

- 如果你调用`job.cancel`然后调用`job.join`，协程将挂起直到job完成。
- 在`job.join`之后调用`job.cancel`没有效果，因为job已经完成。

当你对协程的结果感兴趣时，你使用`Deferred`。当协程完成时，这个结果由`Deferred.await`返回。`Deferred`是一种`Job`类型，它也可以被取消。

在被`cancel`的deferred上调用`await`会抛出`JobCancellationException`。

```kotlin
val deferred = async { … }
deferred.cancel()
val result = deferred.await() // 抛出JobCancellationException!
```

这就是我们得到异常的原因：`await`的作用是挂起协程直到计算出结果；由于协程被取消，无法计算结果。因此，在cancel之后调用await会导致`JobCancellationException: Job was cancelled`。

另一方面，如果你在`deferred.await`之后调用`deferred.cancel`，什么也不会发生，因为协程已经完成。

## 处理取消副作用

假设你想在协程被取消时执行特定操作：关闭你可能使用的任何资源、记录取消或执行一些其他清理代码。有几种方法可以做到这一点：

### 检查!isActive

如果你定期检查`isActive`，那么一旦退出while循环，你就可以清理资源。我们上面的代码可以更新为：

```kotlin
val job = scope.launch {
    var nextPrintTime = startTime
    var i = 0
    while (i < 5 && isActive) {
        // 每半秒打印一次消息
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("Hello ${i++}")
            nextPrintTime += 500L
        }
    }
    // 协程工作完成，所以我们可以清理
    println("Clean up!")
}
```

现在，当协程不再活跃时，`while`将中断，我们可以进行清理。

### Try catch finally

由于协程取消时会抛出`CancellationException`，我们可以将挂起工作包装在`try/catch`中，在`finally`块中实现清理工作。

```kotlin
val job = launch {
    try {
        work()
    } catch (e: CancellationException){
        println("Work cancelled!")
    } finally {
        println("Clean up!")
    }
}
delay(1000L)
println("Cancel!")
job.cancel()
println("Done!")
```

但是，如果我们需要执行的清理工作是挂起的，上面的代码将不再工作，因为一旦协程处于取消状态，它就不能再挂起。

> 处于取消状态的协程无法挂起！

为了能够在协程被取消时调用`suspend`函数，我们需要将需要做的清理工作切换到`NonCancellable` `CoroutineContext`中。这将允许代码挂起，并将保持协程处于*取消中*状态，直到工作完成。

```kotlin
val job = launch {
    try {
        work()
    } catch (e: CancellationException){
        println("Work cancelled!")
    } finally {
        withContext(NonCancellable) {
            delay(1000L) // 或其他挂起函数
            println("Cleanup done!")
        }
    }
}
delay(1000L)
println("Cancel!")
job.cancel()
println("Done!")
```

这种方法允许你在协程被取消后执行必要的清理工作，即使这些清理工作需要挂起函数。

## suspendCancellableCoroutine和invokeOnCancellation

如果你使用`suspendCoroutine`方法将回调转换为协程，那么最好使用`suspendCancellableCoroutine`。可以使用`continuation.invokeOnCancellation`实现取消时要完成的工作：

```kotlin
suspend fun work() {
    return suspendCancellableCoroutine { continuation ->
        continuation.invokeOnCancellation {
            // 执行清理工作
        }
        // 实现的其余部分
    }
}
```

这种方法在桥接传统的回调API和协程时特别有用，确保当协程被取消时，相应的回调或资源也能得到适当的处理。

## 协作式取消的最佳实践

1. **定期检查取消状态**：
   - 在长时间运行的循环中使用`isActive`或`ensureActive()`
   - 在CPU密集型任务中使用`yield()`

2. **使用可取消的挂起函数**：
   - 优先使用`kotlinx.coroutines`库中的挂起函数
   - 这些函数内置了取消检查机制

3. **正确处理清理工作**：
   - 使用`try-finally`块进行资源清理
   - 对于需要挂起的清理工作，使用`withContext(NonCancellable)`

4. **及时响应取消**：
   - 避免长时间阻塞操作
   - 将大任务分解为可中断的小任务

```kotlin
// 良好的协作式取消示例
suspend fun processLargeDataSet(data: List<Data>) {
    data.forEach { item ->
        ensureActive() // 检查取消状态
        processItem(item)
        yield() // 让出执行权
    }
}
```

## 总结

要实现结构化并发的好处并确保我们不做不必要的工作，你需要确保你的代码也是可取消的。

关键要点：

- **取消是协作式的**：协程代码需要主动检查和响应取消请求
- **使用合适的作用域**：使用Jetpack提供的`viewModelScope`或`lifecycleScope`
- **正确处理取消副作用**：使用适当的机制处理资源清理
- **选择合适的取消检查方法**：根据具体场景选择`isActive`、`ensureActive()`或`yield()`

协程的取消机制为我们提供了强大的工具来管理应用程序中的异步工作，确保资源得到有效利用，避免不必要的计算和内存占用。

---

**下一步学习：**
- [第3部分：异常和监督](https://medium.com/androiddevelopers/exceptions-and-supervision-in-coroutines-1cb51b4b24a5)
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

*本文翻译自 Android Developers 官方博客，更多内容请关注 Android 开发者资源。* 