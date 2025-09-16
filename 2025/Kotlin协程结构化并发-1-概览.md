# Kotlin协程结构化并发-1-概览

> 协程中的取消和异常处理 - 第1部分

*本系列博客文章深入探讨协程中的取消和异常处理。取消对于避免执行不必要的工作很重要，这可以节省内存和电池寿命；正确的异常处理是良好用户体验的关键。作为本系列其他2部分（第2部分：取消，第3部分：异常）的基础，定义一些核心协程概念很重要，如`CoroutineScope`、`Job`和`CoroutineContext`，以确保我们都在同一页面上。*

**系列文章目录：**
- [第1部分：协程基础概念](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)
- [第2部分：取消和异常](https://medium.com/androiddevelopers/cancellation-and-exceptions-in-coroutines-b5403f4d4c94)
- [第3部分：异常和监督](https://medium.com/androiddevelopers/exceptions-and-supervision-in-coroutines-1cb51b4b24a5)
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

## CoroutineScope（协程作用域）

`CoroutineScope`用于跟踪通过`launch`或`async`创建的所有协程（这些是`CoroutineScope`上的扩展函数）。正在进行的工作（运行中的协程）可以通过在任何时候调用`scope.cancel()`来取消。

当你想在应用程序的特定层启动和控制协程的生命周期时，应该创建一个`CoroutineScope`。在某些平台如Android中，有KTX库已经在某些生命周期类中提供了`CoroutineScope`，如`viewModelScope`和`lifecycleScope`。

创建`CoroutineScope`时，它需要一个`CoroutineContext`作为构造函数的参数。你可以使用以下代码创建新的作用域和协程：

```kotlin
// Job和Dispatcher组合成一个CoroutineContext，
// 稍后会详细讨论
val scope = CoroutineScope(Job() + Dispatchers.Main)

val job = scope.launch {
    // 新的协程
}
```

## Job（作业）

`Job`是协程的句柄。对于你创建的每个协程（通过`launch`或`async`），它都会返回一个`Job`实例，该实例唯一标识协程并管理其生命周期。如上所示，你也可以将`Job`传递给`CoroutineScope`以保持对其生命周期的控制。

## CoroutineContext（协程上下文）

`CoroutineContext`是定义协程行为的一组元素。它由以下部分组成：

- **Job** — 控制协程的生命周期
- **CoroutineDispatcher** — 将工作分发到适当的线程
- **CoroutineName** — 协程的名称，用于调试
- **CoroutineExceptionHandler** — 处理未捕获的异常，将在系列的第3部分中介绍

新协程的`CoroutineContext`是什么？我们已经知道会创建一个新的`Job`实例，让我们能够控制其生命周期。其余元素将从其父级的`CoroutineContext`继承（父级可以是另一个协程或创建它的`CoroutineScope`）。

由于`CoroutineScope`可以创建协程，并且你可以在协程内部创建更多协程，因此会创建一个隐式的**任务层次结构**。在以下代码片段中，除了使用`CoroutineScope`创建新协程外，看看如何在协程内部创建更多协程：

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main)

val job = scope.launch {
    // 新协程，以CoroutineScope作为父级
    val result = async {
        // 新协程，以launch启动的协程作为父级
    }.await()
}
```

该层次结构的根通常是`CoroutineScope`。我们可以将该层次结构可视化如下：

```
CoroutineScope
    |
    └── launch协程
            |
            └── async协程
```

*协程在任务层次结构中执行。父级可以是CoroutineScope或另一个协程。*

## Job生命周期

`Job`可以经历一系列状态：New（新建）、Active（活跃）、Completing（完成中）、Completed（已完成）、Cancelling（取消中）和Cancelled（已取消）。虽然我们无法直接访问状态本身，但可以访问Job的属性：`isActive`、`isCancelled`和`isCompleted`。

```
New ──────────────► Active ──────────► Completing ────► Completed
                      |                                       ▲
                      |                                       |
                      └─────► Cancelling ─────────────────────┘
                                  |
                                  ▼
                              Cancelled
```

**Job生命周期状态说明：**

- **New（新建）**：Job刚创建但尚未启动
- **Active（活跃）**：Job正在运行中
- **Completing（完成中）**：Job的执行已完成，等待子Job完成
- **Completed（已完成）**：Job及其所有子Job都已完成
- **Cancelling（取消中）**：Job正在取消过程中
- **Cancelled（已取消）**：Job已被取消

如果协程处于活跃状态，协程失败或调用`job.cancel()`将使job进入Cancelling状态（`isActive = false`，`isCancelled = true`）。一旦所有子协程完成其工作，协程将进入Cancelled状态，并且`isCompleted = true`。

## 父CoroutineContext详解

在任务层次结构中，每个协程都有一个父级，它可以是`CoroutineScope`或另一个协程。然而，协程的结果父`CoroutineContext`可能与父级的`CoroutineContext`不同，因为它是基于以下公式计算的：

> **父上下文** = 默认值 + 继承的`CoroutineContext` + 参数

其中：
- 某些元素具有**默认**值：`Dispatchers.Default`是`CoroutineDispatcher`的默认值，`"coroutine"`是`CoroutineName`的默认值
- **继承的**`CoroutineContext`是创建它的`CoroutineScope`或协程的`CoroutineContext`
- 协程构建器中传递的**参数**将优先于继承上下文中的那些元素

**注意**：`CoroutineContext`可以使用`+`操作符组合。由于`CoroutineContext`是一组元素，将创建一个新的`CoroutineContext`，其中加号右侧的元素会覆盖左侧的元素。例如：`(Dispatchers.Main, "name") + (Dispatchers.IO) = (Dispatchers.IO, "name")`

### CoroutineContext组合示例

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main + CoroutineName("ParentScope"))

// 这个协程的父CoroutineContext包含：
// - Job (从scope继承)
// - Dispatchers.Main (从scope继承)
// - CoroutineName("ParentScope") (从scope继承)
// - 默认值会填充其他未指定的元素

val job = scope.launch(Dispatchers.IO + CoroutineName("ChildCoroutine")) {
    // 这个新协程的实际CoroutineContext包含：
    // - 新的Job实例 (总是创建新的)
    // - Dispatchers.IO (参数覆盖了继承的Main)
    // - CoroutineName("ChildCoroutine") (参数覆盖了继承的名称)
}
```

*此CoroutineScope启动的每个协程在CoroutineContext中至少会有这些元素。CoroutineName显示为灰色，因为它来自默认值。*

现在我们知道了新协程的父`CoroutineContext`是什么，它的实际`CoroutineContext`将是：

> **新协程上下文** = 父`CoroutineContext` + `Job()`

如果使用上图所示的`CoroutineScope`创建一个新协程，如下所示：

```kotlin
val job = scope.launch(Dispatchers.IO) {
    // 新协程
}
```

该协程的父`CoroutineContext`和实际`CoroutineContext`是什么？

**结果分析：**
- **父CoroutineContext**：Job（来自scope）+ Dispatchers.IO（参数覆盖）+ CoroutineName（默认值）
- **实际CoroutineContext**：新Job实例 + Dispatchers.IO + CoroutineName

*CoroutineContext中的Job和父上下文中的Job永远不会是同一个实例，因为新协程总是获得Job的新实例*

结果父`CoroutineContext`具有`Dispatchers.IO`而不是作用域的`CoroutineDispatcher`，因为它被协程构建器的参数覆盖了。另外，请注意父`CoroutineContext`中的`Job`是作用域`Job`的实例（红色），而新协程的实际`CoroutineContext`被分配了一个新的`Job`实例（绿色）。

## SupervisorJob特殊情况

正如我们将在本系列的第3部分中看到的，`CoroutineScope`在其`CoroutineContext`中可以有一个不同的`Job`实现，称为`SupervisorJob`，这改变了`CoroutineScope`处理异常的方式。因此，使用该作用域创建的新协程可以将`SupervisorJob`作为父`Job`。但是，当协程的父级是另一个协程时，父`Job`总是`Job`类型。

## 结构化并发原则

协程遵循**结构化并发**原则，这意味着：

1. **层次结构管理**：协程按层次结构组织，每个协程都有明确的父子关系
2. **生命周期绑定**：子协程的生命周期与父协程绑定
3. **异常传播**：异常会在层次结构中传播
4. **取消传播**：取消操作会传播给所有子协程

```kotlin
// 结构化并发示例
val scope = CoroutineScope(Job() + Dispatchers.Main)

scope.launch {
    // 父协程
    launch {
        // 子协程1
        delay(1000)
        println("子协程1完成")
    }
    
    launch {
        // 子协程2
        delay(2000)
        println("子协程2完成")
    }
    
    // 父协程会等待所有子协程完成
    println("所有子协程完成后，父协程才会完成")
}
```

## 最佳实践

1. **使用合适的作用域**：
   - 在Android中使用`viewModelScope`或`lifecycleScope`
   - 对于应用级别的工作，创建自定义的application scope

2. **正确处理生命周期**：
   - 在适当的时候取消作用域
   - 避免内存泄漏

3. **异常处理**：
   - 使用`CoroutineExceptionHandler`处理未捕获的异常
   - 在适当的地方使用try-catch

4. **测试友好**：
   - 注入`CoroutineDispatcher`以便测试
   - 使用`TestDispatcher`进行单元测试

## 总结

现在你了解了协程的基础知识，可以开始学习更多关于协程中的取消和异常处理：

- **CoroutineScope**：管理协程生命周期的作用域
- **Job**：协程的句柄，用于控制单个协程
- **CoroutineContext**：定义协程行为的上下文信息
- **结构化并发**：协程的层次化管理原则

这些概念为理解协程的高级特性（如取消、异常处理等）奠定了基础。

---

**下一步学习：**
- [第2部分：取消和异常](https://medium.com/androiddevelopers/cancellation-and-exceptions-in-coroutines-b5403f4d4c94)
- [第3部分：异常和监督](https://medium.com/androiddevelopers/exceptions-and-supervision-in-coroutines-1cb51b4b24a5)  
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

*本文翻译自 Android Developers 官方博客，更多内容请关注 Android 开发者资源。* 