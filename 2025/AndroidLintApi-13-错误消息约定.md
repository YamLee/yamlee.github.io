# Android Lint API 指南 - 错误消息约定

## 错误消息约定

编写清晰、有用的错误消息是创建高质量Lint检查的重要部分。好的错误消息应该能够帮助开发者快速理解问题并知道如何修复它。

### 基本原则

#### 1. 清晰简洁

错误消息应该直接说明问题，避免技术术语和冗长的解释：

```kotlin
// 好的消息
"Missing contentDescription attribute on ImageView"

// 不好的消息  
"The ImageView element lacks the android:contentDescription attribute which is required for accessibility support and should be provided to describe the image content for screen readers and other assistive technologies"
```

#### 2. 可操作性

消息应该告诉开发者如何修复问题：

```kotlin
// 好的消息
"Replace `getColor()` with `ContextCompat.getColor()`"

// 不好的消息
"Deprecated API usage detected"
```

#### 3. 具体性

提供具体的信息而不是泛泛而谈：

```kotlin
// 好的消息
"Method `onCreate()` is missing call to `super.onCreate()`"

// 不好的消息
"Missing required method call"
```

### 消息结构

#### 标准格式

推荐的错误消息格式：

```
[问题描述]: [具体建议]
```

示例：

```kotlin
class MessageExampleDetector : Detector(), Detector.UastScanner {
    
    companion object {
        val ISSUE = Issue.create(
            id = "MissingSuper",
            briefDescription = "Missing call to super",
            explanation = """
                Most lifecycle methods should call through to the super implementation. 
                This ensures that the framework can perform necessary initialization.
                """,
            category = Category.CORRECTNESS,
            priority = 9,
            severity = Severity.ERROR,
            implementation = Implementation(
                MessageExampleDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
    
    override fun visitMethod(context: JavaContext, node: UMethod) {
        if (node.name == "onCreate" && !hasSuperCall(node)) {
            context.report(
                ISSUE,
                node,
                context.getNameLocation(node),
                "Missing call to `super.onCreate()` in lifecycle method"
            )
        }
    }
}
```

### 消息类型

#### 1. 错误消息

用于必须修复的问题：

```kotlin
// API使用错误
"Call to deprecated method `getColor(int)`. Use `ContextCompat.getColor()` instead"

// 安全问题
"Potential SQL injection: parameter `$paramName` is not sanitized"

// 必需属性缺失
"Missing required attribute `android:contentDescription`"
```

#### 2. 警告消息

用于建议改进的问题：

```kotlin
// 性能建议
"Consider using `RecyclerView` instead of `ListView` for better performance"

// 代码风格
"Method name `$methodName` should be in camelCase"

// 最佳实践
"Consider making this field `private` and providing accessor methods"
```

#### 3. 信息消息

用于提供有用信息：

```kotlin
// 代码度量
"Method has ${lineCount} lines (consider breaking it down)"

// 使用统计
"This deprecated API is used ${usageCount} times in your project"

// 建议
"Consider using Kotlin instead of Java for new files"
```

### 动态消息内容

#### 使用变量

在消息中包含具体的变量值：

```kotlin
class DynamicMessageDetector : Detector(), Detector.UastScanner {
    
    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        val methodName = method.name
        val className = method.containingClass?.name
        val argumentCount = node.valueArguments.size
        
        context.report(
            ISSUE,
            node,
            context.getLocation(node),
            "Method `$className.$methodName()` called with $argumentCount arguments, expected 2"
        )
    }
}
```

#### 条件消息

根据不同情况显示不同消息：

```kotlin
class ConditionalMessageDetector : Detector(), Detector.UastScanner {
    
    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        val apiLevel = getRequiredApiLevel(method)
        val minSdk = context.mainProject.minSdk
        
        val message = when {
            apiLevel > minSdk -> 
                "This method requires API level $apiLevel, minimum is $minSdk"
            apiLevel == minSdk ->
                "This method requires API level $apiLevel (same as minimum SDK)"
            else ->
                "This method is available since API level $apiLevel"
        }
        
        context.report(ISSUE, node, context.getLocation(node), message)
    }
}
```

### 多语言支持

虽然Lint主要使用英文，但可以为错误消息添加本地化支持：

```kotlin
class LocalizedMessageDetector : Detector(), Detector.UastScanner {
    
    companion object {
        private val messages = mapOf(
            "en" to "Missing required permission",
            "zh" to "缺少必需的权限",
            "ja" to "必要な権限がありません",
            "es" to "Falta el permiso requerido"
        )
    }
    
    private fun getLocalizedMessage(context: JavaContext, key: String): String {
        val locale = context.project.locale ?: "en"
        return messages[locale] ?: messages["en"]!!
    }
}
```

### 消息格式化

#### 代码高亮

使用反引号突出显示代码元素：

```kotlin
"Replace `findViewById()` with view binding or `ViewCompat.requireViewById()`"
"The method `$methodName` should be `private`"
"Use `@NonNull` annotation on parameter `$paramName`"
```

#### 列表格式

对于多个建议，使用清晰的格式：

```kotlin
val suggestions = listOf(
    "Use `RecyclerView` instead of `ListView`",
    "Implement `ViewHolder` pattern",
    "Use `DiffUtil` for efficient updates"
)

val message = "Consider these performance improvements:\n" + 
    suggestions.joinToString("\n") { "• $it" }
```

### 错误位置

#### 精确定位

尽可能精确地指出问题位置：

```kotlin
class PreciseLocationDetector : Detector(), Detector.UastScanner {
    
    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        val problemArgument = node.valueArguments.firstOrNull { isProblem(it) }
        
        if (problemArgument != null) {
            // 指向具体的参数而不是整个方法调用
            context.report(
                ISSUE,
                problemArgument,
                context.getLocation(problemArgument),
                "This argument should not be null"
            )
        }
    }
}
```

#### 范围选择

对于跨越多行的问题，选择合适的范围：

```kotlin
// 对于方法签名问题，选择方法名
context.report(ISSUE, node, context.getNameLocation(node), message)

// 对于整个语句问题，选择整个节点
context.report(ISSUE, node, context.getLocation(node), message)

// 对于特定表达式问题，选择该表达式
context.report(ISSUE, expression, context.getLocation(expression), message)
```

### 消息分类

#### 按严重性分类

```kotlin
// 致命错误：会导致崩溃或编译失败
"Syntax error: missing closing parenthesis"

// 错误：违反API约定或最佳实践
"Missing required attribute `android:layout_width`"

// 警告：潜在问题或改进建议
"This method is deprecated, consider using the newer alternative"

// 信息：代码度量或统计信息
"This class has ${methodCount} public methods"
```

#### 按类别分类

```kotlin
// 安全问题
"Potential security vulnerability: hardcoded password"

// 性能问题  
"Inefficient operation in loop, consider caching the result"

// 可访问性问题
"Missing accessibility support for users with disabilities"

// 国际化问题
"Hardcoded text should be extracted to string resources"
```

### 最佳实践示例

```kotlin
class BestPracticeDetector : Detector(), Detector.UastScanner {
    
    companion object {
        val INEFFICIENT_LOOP = Issue.create(
            id = "InefficientLoop",
            briefDescription = "Inefficient loop operation",
            explanation = """
                This operation is being performed inside a loop, which can be expensive.
                Consider moving the operation outside the loop or caching the result.
                """,
            category = Category.PERFORMANCE,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                BestPracticeDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
    
    override fun visitForEachExpression(
        context: JavaContext,
        node: UForEachExpression
    ) {
        node.body?.accept(object : AbstractUastVisitor() {
            override fun visitCallExpression(node: UCallExpression): Boolean {
                if (isExpensiveOperation(node)) {
                    val operationName = node.methodName
                    val suggestion = getSuggestion(node)
                    
                    context.report(
                        INEFFICIENT_LOOP,
                        node,
                        context.getLocation(node),
                        "Expensive operation `$operationName()` called in loop. $suggestion"
                    )
                }
                return super.visitCallExpression(node)
            }
        })
    }
    
    private fun getSuggestion(node: UCallExpression): String {
        return when (node.methodName) {
            "findViewById" -> "Consider using view binding or caching the view reference"
            "getSystemService" -> "Cache the service reference outside the loop"
            "getSharedPreferences" -> "Cache the preferences instance"
            else -> "Consider optimizing this operation"
        }
    }
}
```

### 错误消息测试

确保您的错误消息清晰有用：

```kotlin
class MessageTest {
    
    @Test
    fun testErrorMessage() {
        lint()
            .files(
                java("""
                    public class Test {
                        public void onCreate() {
                            // Missing super call
                        }
                    }
                """).indented()
            )
            .run()
            .expect("""
                src/Test.java:2: Error: Missing call to super.onCreate() in lifecycle method [MissingSuper]
                    public void onCreate() {
                                ~~~~~~~~
                1 errors, 0 warnings
            """.trimIndent())
    }
}
```

### 总结

好的错误消息应该：

1. **简洁明了**：直接说明问题
2. **可操作**：告诉开发者如何修复
3. **具体**：提供准确的信息
4. **有帮助**：包含有用的建议
5. **一致**：遵循统一的格式和风格

通过遵循这些约定，您可以创建更好的用户体验，帮助开发者更快地理解和修复代码问题。 