# Kotlin协程结构化并发-3-协程异常处理

> 协程中的取消和异常处理 - 第3部分 - 全面掌握异常处理！

*我们开发者通常花费大量时间完善应用程序的正常路径。然而，当事情不如预期时，提供适当的用户体验同样重要。一方面，看到应用程序崩溃对用户来说是糟糕的体验；另一方面，当操作没有成功时向用户显示正确的消息是必不可少的。*

*正确处理异常对用户如何看待你的应用程序有巨大影响。在这篇文章中，我们将解释异常如何在协程中传播，以及如何始终保持控制，包括处理异常的不同方法。*

> ⚠️ 为了顺利阅读本文的其余部分，需要先阅读并理解本系列的第1部分。

**系列文章目录：**
- [第1部分：协程基础概念](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)
- [第2部分：协程的取消](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629)
- [第3部分：协程异常处理](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

## 协程突然失败了！现在怎么办？😱

当协程因异常失败时，它会将该异常向上传播给其父级！然后，父级会：1）取消其余子协程，2）取消自己，3）将异常向上传播给其父级。

异常将到达层次结构的根部，`CoroutineScope`启动的所有协程也会被取消。

```
CoroutineScope
    |
    └── parent 协程
            |
            ├── child 协程1 (💥异常发生)
            └── child 协程2 (❌被取消)
```

*协程中的异常将在协程层次结构中传播*

虽然传播异常在某些情况下是有意义的，但在其他情况下这是不希望的。想象一个处理用户交互的与UI相关的`CoroutineScope`。如果子协程抛出异常，UI作用域将被取消，整个UI组件将变得无响应，因为已取消的作用域无法启动更多协程。

如果你不希望这种行为怎么办？或者，你可以在创建这些协程的`CoroutineScope`的`CoroutineContext`中使用`Job`的不同实现，即`SupervisorJob`。

## SupervisorJob来拯救

使用`SupervisorJob`，子协程的失败不会影响其他子协程。`SupervisorJob`不会取消自己或其余子协程。此外，`SupervisorJob`也不会传播异常，而是让子协程处理它。

你可以像这样创建一个`CoroutineScope`：`val uiScope = CoroutineScope(SupervisorJob())`，以便在协程失败时不传播取消，如这张图所示：

```
SupervisorScope
    |
    ├── child 协程1 (💥异常发生，不影响其他协程)
    └── child 协程2 (✅继续正常运行)
```

*SupervisorJob不会取消自己或其余子协程*

如果异常未被处理且`CoroutineContext`没有`CoroutineExceptionHandler`（我们稍后会看到），它将到达默认线程的`ExceptionHandler`。在JVM中，异常将被记录到控制台；在Android中，无论发生在哪个Dispatcher上，它都会使你的应用程序崩溃。

> 💥 无论你使用哪种类型的Job，未捕获的异常总是会被抛出

同样的行为适用于作用域构建器`coroutineScope`和`supervisorScope`。这些将创建一个子作用域（相应地将`Job`或`SupervisorJob`作为父级），你可以用它逻辑地分组协程（例如，如果你想进行并行计算或你希望它们受或不受彼此影响）。

**警告**：`SupervisorJob`**只有**在作为作用域的一部分时才能按描述的那样工作：要么使用`supervisorScope`创建，要么使用`CoroutineScope(SupervisorJob())`创建。

## Job还是SupervisorJob？🤔

何时应该使用`Job`或`SupervisorJob`？当你不希望失败取消父级和兄弟协程时，使用`SupervisorJob`或`supervisorScope`。

一些例子：

```kotlin
// 处理应用程序特定层协程的作用域
val scope = CoroutineScope(SupervisorJob())

scope.launch {
    // Child 1
}

scope.launch {
    // Child 2  
}
```

在这种情况下，如果`child#1`失败，scope和`child#2`都**不会**被取消。

另一个例子：

```kotlin
// 处理应用程序特定层协程的作用域
val scope = CoroutineScope(Job())

scope.launch {
    supervisorScope {
        launch {
            // Child 1
        }
        launch {
            // Child 2
        }
    }
}
```

在这种情况下，由于`supervisorScope`创建了一个带有`SupervisorJob`的子作用域，如果`child#1`失败，`child#2`**不会**被取消。如果你在实现中使用`coroutineScope`，失败将传播并最终也会取消scope。

## 小测验：谁是我的父级？🎯

给定以下代码片段，你能识别出`child#1`的父级是什么类型的`Job`吗？

```kotlin
val scope = CoroutineScope(Job())

scope.launch(SupervisorJob()) {
    // 新协程 -> 可以挂起
    launch {
        // Child 1
    }
    launch {
        // Child 2
    }
}
```

`child#1`的parentJob是`Job`类型！希望你答对了！尽管乍一看，你可能认为它可以是`SupervisorJob`，但它不是，因为新协程总是被分配一个新的`Job()`，在这种情况下它会覆盖`SupervisorJob`。`SupervisorJob`是用`scope.launch`创建的协程的父级；所以从字面上说，`SupervisorJob`在那段代码中什么都不做！

```
scope
  |
  └── launch协程 (parent: SupervisorJob)
          |
          ├── child#1 (parent: Job) ❌
          └── child#2 (parent: Job) ❌
```

*child#1和child#2的父级是Job类型，而不是SupervisorJob*

因此，如果`child#1`或`child#2`中的任何一个失败，失败将到达scope，该scope启动的所有工作都将被取消。

记住，`SupervisorJob`**只有**在作为作用域的一部分时才能按描述的那样工作：要么使用`supervisorScope`创建，要么使用`CoroutineScope(SupervisorJob())`创建。将`SupervisorJob`作为协程构建器的参数传递不会产生你对取消所期望的效果。

关于异常，如果任何子协程抛出异常，那个`SupervisorJob`不会在层次结构中向上传播异常，而是让其协程处理它。

## 底层实现

如果你对`Job`在底层如何工作感到好奇，请查看`JobSupport.kt`文件中`childCancelled`和`notifyCancelling`函数的实现。

在`SupervisorJob`实现中，`childCancelled`方法只返回`false`，这意味着它不传播取消，但也不处理异常。

## 处理异常👩‍🚒

协程使用常规的Kotlin语法来处理异常：`try/catch`或内置助手函数如`runCatching`（内部使用`try/catch`）。

我们之前说过**未捕获的异常总是会被抛出**。然而，不同的协程构建器以不同的方式处理异常。

### Launch

使用`launch`，**异常将在发生时立即抛出**。因此，你可以将可能抛出异常的代码包装在`try/catch`中，如此例所示：

```kotlin
scope.launch {
    try {
        codeThatCanThrowExceptions()
    } catch(e: Exception) {
        // 处理异常
    }
}
```

> 使用launch，异常将在发生时立即抛出

### Async

当`async`用作根协程（是`CoroutineScope`实例或`supervisorScope`的直接子协程）时，**异常不会自动抛出，而是在你调用`.await()`时抛出**。

要处理`async`作为根协程时抛出的异常，你可以将`.await()`调用包装在`try/catch`中：

```kotlin
supervisorScope {
    val deferred = async {
        codeThatCanThrowExceptions()
    }
    
    try {
        deferred.await()
    } catch(e: Exception) {
        // 处理在async中抛出的异常
    }
}
```

在这种情况下，请注意调用`async`永远不会抛出异常，这就是为什么不需要也包装它。`await`将抛出在`async`协程内部发生的异常。

> 当async用作根协程时，异常在你调用.await时抛出

另外，请注意我们使用`supervisorScope`来调用`async`和`await`。正如我们之前所说，`SupervisorJob`让协程处理异常；与`Job`相反，`Job`会自动在层次结构中向上传播异常，因此`catch`块不会被调用：

```kotlin
coroutineScope {
    try {
        val deferred = async {
            codeThatCanThrowExceptions()
        }
        deferred.await()
    } catch(e: Exception) {
        // 在async中抛出的异常不会在这里被捕获
        // 而是向上传播到作用域
    }
}
```

此外，在其他协程创建的协程中发生的异常将始终传播，无论使用何种协程构建器。例如：

```kotlin
val scope = CoroutineScope(Job())

scope.launch {
    async {
        // 如果async抛出异常，launch会抛出异常而不调用.await()
    }
}
```

在这种情况下，如果`async`抛出异常，它将在发生时立即被抛出，因为作用域的直接子协程是`launch`。原因是`async`（在其`CoroutineContext`中有一个`Job`）将自动向上传播异常到其父级（`launch`），而`launch`会抛出异常。

> ⚠️ 在coroutineScope构建器中或在其他协程创建的协程中抛出的异常不会在try/catch中被捕获！

在`SupervisorJob`部分，我们提到了`CoroutineExceptionHandler`的存在。让我们深入了解它！

## CoroutineExceptionHandler

`CoroutineExceptionHandler`是`CoroutineContext`的可选元素，允许你**处理未捕获的异常**。

以下是如何定义`CoroutineExceptionHandler`，每当捕获异常时，你都有关于发生异常的`CoroutineContext`和异常本身的信息：

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    println("Caught $exception")
}
```

如果满足以下要求，异常将被捕获：

- **何时**⏰：异常由_自动抛出异常_的协程抛出（适用于`launch`，不适用于`async`）。
- **何地**🌍：如果它在`CoroutineScope`或根协程（`CoroutineScope`或`supervisorScope`的直接子协程）的`CoroutineContext`中。

让我们看一些使用上面定义的`CoroutineExceptionHandler`的例子。在以下例子中，异常**会**被处理器捕获：

```kotlin
val scope = CoroutineScope(Job())

scope.launch(handler) {
    launch {
        throw Exception("Failed coroutine")
    }
}
```

在另一种情况下，处理器安装在内部协程中，它**不会**被捕获：

```kotlin
val scope = CoroutineScope(Job())

scope.launch {
    launch(handler) {
        throw Exception("Failed coroutine")
    }
}
```

异常不会被捕获，因为处理器没有安装在正确的`CoroutineContext`中。内部的`launch`会立即将异常向上传播给父级，由于父级对处理器一无所知，异常将被抛出。

## 异常处理最佳实践

### 1. 选择合适的Job类型

```kotlin
// 当你希望一个子协程的失败不影响其他子协程时
val scope = CoroutineScope(SupervisorJob() + handler)

// 当你希望任何子协程的失败都取消整个作用域时
val scope = CoroutineScope(Job() + handler)
```

### 2. 正确处理async异常

```kotlin
// ✅ 正确的方式
supervisorScope {
    val deferred = async { riskyOperation() }
    try {
        val result = deferred.await()
        // 使用结果
    } catch (e: Exception) {
        // 处理异常
    }
}

// ❌ 错误的方式 - 异常不会被捕获
coroutineScope {
    try {
        val result = async { riskyOperation() }.await()
    } catch (e: Exception) {
        // 这里不会捕获到异常
    }
}
```

### 3. 使用CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("CoroutineException", "Caught $exception")
    // 上报错误到崩溃分析系统
    // 显示用户友好的错误消息
}

val scope = CoroutineScope(SupervisorJob() + handler)

scope.launch {
    // 这里的未捕获异常会被handler处理
    throw RuntimeException("Something went wrong")
}
```

### 4. 混合使用不同的策略

```kotlin
class UserRepository {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    
    suspend fun fetchUserData(userId: String): Result<User> {
        return try {
            val userData = scope.async {
                // 网络请求获取用户数据
                apiService.getUser(userId)
            }
            Result.success(userData.await())
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    fun preloadData() {
        scope.launch {
            try {
                // 预加载数据，失败不应该影响其他操作
                preloadCache()
            } catch (e: Exception) {
                // 记录错误但不影响应用运行
                Log.w("Repository", "Preload failed", e)
            }
        }
    }
}
```

## 实际应用场景

### Android ViewModel中的异常处理

```kotlin
class MyViewModel : ViewModel() {
    private val handler = CoroutineExceptionHandler { _, exception ->
        // 处理未捕获的异常
        _errorState.value = exception.message ?: "Unknown error"
    }
    
    private val vmScope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
    
    fun loadData() {
        vmScope.launch {
            try {
                val data = withContext(Dispatchers.IO) {
                    repository.fetchData()
                }
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
    
    override fun onCleared() {
        vmScope.cancel()
        super.onCleared()
    }
}
```

### 并发任务的异常处理

```kotlin
suspend fun processMultipleTasks() = supervisorScope {
    val tasks = listOf("task1", "task2", "task3")
    
    val results = tasks.map { taskName ->
        async {
            try {
                processTask(taskName)
            } catch (e: Exception) {
                Log.w("Task", "Task $taskName failed: ${e.message}")
                null // 返回默认值或null
            }
        }
    }.awaitAll()
    
    // 过滤掉失败的任务结果
    results.filterNotNull()
}
```

## 总结

在应用程序中优雅地处理异常对于拥有良好的用户体验很重要，即使在事情不如预期时也是如此。

**关键要点：**

1. **异常传播**：协程中的异常会在层次结构中向上传播，直到被处理或到达根作用域
2. **Job vs SupervisorJob**：
   - 使用`Job`当你希望异常传播并取消整个作用域
   - 使用`SupervisorJob`当你希望避免异常传播取消
3. **launch vs async**：
   - `launch`中的异常立即抛出
   - `async`中的异常在调用`await()`时抛出
4. **CoroutineExceptionHandler**：用于处理未捕获的异常
5. **最佳实践**：结合使用不同的异常处理策略，根据业务需求选择合适的方法

记住使用`SupervisorJob`当你希望在发生异常时避免传播取消，否则使用`Job`。

未捕获的异常将传播，捕获它们以提供出色的用户体验！

---

**下一步学习：**
- [第4部分：不应该被取消的工作模式](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

*本文翻译自 [Android Developers 官方博客](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)，更多内容请关注 Android 开发者资源。* 