# 不应该被取消的工作的协程模式

> 协程中的取消和异常 - 第4部分

*本文是探索协程中取消和异常处理系列文章的第4部分。你可以查看本系列的其他文章：*
- [第1部分：协程中的异常](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
- [第2部分：取消和异常](https://medium.com/androiddevelopers/cancellation-and-exceptions-in-coroutines-b5403f4d4c94)
- [第3部分：异常和监督](https://medium.com/androiddevelopers/exceptions-and-supervision-in-coroutines-1cb51b4b24a5)

在某些情况下，你需要执行一些不应该被取消的重要工作，即使是在用户离开屏幕时。在这种情况下，将工作**移出UI**并在**应用级别**的`CoroutineScope`中运行它是一个好的做法。

在这篇文章中，我将解释何时以及如何操作，并提供不同方法的替代方案。

## 协程 vs WorkManager

在我们开始之前，重要的是要区分何时使用**协程**以及何时使用[**WorkManager**](https://developer.android.com/topic/libraries/architecture/workmanager)来进行不应该被取消的工作。

**WorkManager**是为保证执行而设计的，即使应用程序退出或设备重新启动也是如此。WorkManager将持久化工作并在以后重新启动。协程默认情况下不会这样做。协程遵循结构化并发原则，在父取消时会被取消。

基于此，以下是使用每种方法的推荐场景：

**WorkManager用于:**
- 应该**保证运行的工作**，即使用户离开应用程序或重新启动设备
- 可以**推迟**的工作

**协程用于:**
- 应该在应用程序运行时**继续运行的工作**
- **不能推迟**的工作

> 💡 如果你有疑问，请使用WorkManager。

## 协程最佳实践

目前为止，我们一直在使用`viewModelScope`和`lifecycleScope`来启动协程，但这些作用域绑定到特定的UI组件的生命周期。当用户离开屏幕时，工作会自动取消。

但是，这种自动取消行为有时并不是你想要的。如果工作对用户很重要，不应该因为用户的UI导航而取消，考虑将工作移动到应用级别的作用域。

## 不应该被取消的操作

以下是一些不应该被取消的操作示例：
- **将数据发送到服务器**
- **将多个文件写入磁盘**
- **对大量数据执行一些分析**

让我们看一个具体示例。考虑以下情况，其中`ViewModel`需要在数据发生变化时将其发送到服务器：

```kotlin
class MyViewModel(
    private val repo: Repository
) : ViewModel() {

    fun sendData(data: Data) {
        viewModelScope.launch { 
            repo.sendData(data)
        }
    }
}
```

在上面的代码中，如果用户在网络请求完成之前离开屏幕，`sendData`将被取消，数据将不会发送到服务器。如果用户稍后回到屏幕，他们将看到陈旧的数据。

为了避免这个问题，你应该将工作移动到应用级别的作用域。

## 创建应用级别的作用域

应用级别的作用域应该存在于应用程序的整个生命周期中，不应该被取消，除非进程被杀死。

你可以通过扩展`Application`类并创建一个持有应用级别作用域的属性来创建一个：

```kotlin
class MyApplication : Application() {
    // 使用默认的Dispatcher.Main，不需要其他配置
    val applicationScope = CoroutineScope(SupervisorJob())
}
```

或者，你可以使用依赖注入。例如，使用Hilt：

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopeModule {

    @Provides
    @Singleton
    fun providesCoroutineScope(): CoroutineScope {
        // 使用默认的Dispatcher.Main，不需要其他配置
        return CoroutineScope(SupervisorJob())
    }
}
```

无论哪种方式，你都需要确保这个作用域：
- 使用`SupervisorJob()`，这样一个子协程中的失败不会影响其他子协程
- 默认使用`Dispatcher.Main`，这样你可以在需要时轻松切换上下文

一旦你有了应用级别的作用域，你可以将其传递给需要执行不应该被取消的工作的类。对于前面的示例：

```kotlin
class Repository(
    private val externalScope: CoroutineScope,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    
    fun sendData(data: Data) {
        externalScope.launch {
            withContext(ioDispatcher) {
                // 执行网络请求
            }
        }
    }
}
```

现在，即使用户离开屏幕，数据也会被发送到服务器。

### 另一种方法

如果你不想将应用级别的作用域传递给Repository，你可以在ViewModel中处理这个问题：

```kotlin
class MyViewModel(
    private val repo: Repository,
    private val externalScope: CoroutineScope
) : ViewModel() {

    fun sendData(data: Data) {
        externalScope.launch { 
            repo.sendData(data)
        }
    }
}
```

然而，这种方法的缺点是每次调用`sendData`时，你都需要记住使用外部作用域。

## 测试

为了能够控制测试中的执行，将应用级别的作用域作为依赖项传递。在测试中，你可以创建一个使用`TestDispatcher`的`TestScope`并将其传递：

```kotlin
class RepositoryTest {
    
    @Test
    fun test() = runTest {
        val repository = Repository(
            externalScope = this, // 这将使用TestScope
            ioDispatcher = UnconfinedTestDispatcher()
        )
        
        repository.sendData(fakeData)
        
        // 进行断言...
    }
}
```

## 替代方案

### 使用ProcessLifecycleOwner

你可以使用`ProcessLifecycleOwner`来获取应用程序进程的生命周期，并使用`lifecycleScope`：

```kotlin
class Repository() {
    
    fun sendData(data: Data) {
        ProcessLifecycleOwner.get().lifecycleScope.launch {
            // 执行工作...
        }
    }
}
```

然而，使用这种方法：
- 你无法轻松控制用于测试的`CoroutineDispatcher`
- 你将Android框架代码与Repository耦合

### 使用GlobalScope

**不要使用`GlobalScope`**。这是一个精致的API，有很好的理由：
- **难以测试**：因为它使用`Dispatchers.Unconfined`，你无法在测试中轻松替换它
- **难以取消**：因为它的生命周期与应用程序进程相同，没有简单的方法来取消在其中启动的工作
- **没有内存泄漏保护**：`GlobalScope`不知道你的应用程序组件，因此无法在适当时取消工作

## 要点总结

- 对于**保证执行**（即使应用程序退出），使用**WorkManager**
- 对于**应该在应用程序活跃时继续运行**的工作，将其移动到**应用级别的CoroutineScope**
- 避免使用`GlobalScope`
- 对于测试，将应用级别的作用域作为依赖项传递

这样，你就可以确保重要的工作不会被意外取消，同时保持代码的可测试性和结构化。

---

*这是协程系列文章的第4部分。更多内容请关注Android开发者博客。* 