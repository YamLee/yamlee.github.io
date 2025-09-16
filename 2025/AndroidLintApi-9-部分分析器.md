# Android Lint API 指南 - 第9章：部分分析器


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第9章节"Partial Analysis"的中文翻译版本。

## 部分分析器

部分分析是Lint 7.0中引入的一个重要功能，旨在提高CI/CD环境中的性能。它允许lint在不完整的项目上下文中运行，这对于增量构建和大型项目特别有用。

### 什么是部分分析

部分分析允许lint在以下情况下运行：
- 只有部分源文件可用
- 缺少某些依赖项
- 在增量构建过程中
- 在CI环境中，只分析变更的文件

### 启用部分分析

#### 在测试中启用

```kotlin
@Test
fun testPartialAnalysis() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        // 测试代码
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .partial()  // 启用部分分析
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

#### 在实际使用中启用

```bash
# 通过命令行启用
./gradlew lint --partial

# 通过系统属性启用
./gradlew lint -Dlint.partial=true
```

### 适配部分分析的检测器

#### 检查作用域兼容性

```kotlin
class PartialCompatibleDetector : Detector(), SourceCodeScanner {
    
    companion object {
        val ISSUE = Issue.create(
            id = "PartialCompatibleIssue",
            briefDescription = "Issue that works with partial analysis",
            explanation = "This issue can be detected even with incomplete information",
            category = Category.CORRECTNESS,
            severity = Severity.WARNING,
            implementation = Implementation(
                PartialCompatibleDetector::class.java,
                // 使用单一作用域，确保部分分析兼容性
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
    
    override fun getApplicableMethodNames() = listOf("problematicMethod")
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 这个检查只依赖当前文件的信息，适合部分分析
        context.report(
            Incident(context, ISSUE)
                .message("Problematic method call detected")
                .at(node)
        )
    }
}
```

#### 处理缺失信息

```kotlin
class RobustDetector : Detector(), SourceCodeScanner {
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 在部分分析中，解析可能失败
        val resolved = node.resolve()
        
        if (resolved == null) {
            // 在部分分析中优雅地处理解析失败
            handleUnresolvedCall(context, node)
        } else {
            // 正常处理已解析的调用
            handleResolvedCall(context, node, resolved)
        }
    }
    
    private fun handleUnresolvedCall(context: JavaContext, node: UCallExpression) {
        // 基于可用信息进行最佳猜测
        val methodName = node.methodName
        if (methodName in KNOWN_PROBLEMATIC_METHODS) {
            context.report(
                Incident(context, ISSUE)
                    .message("Potentially problematic method: $methodName")
                    .at(node)
            )
        }
    }
    
    private fun handleResolvedCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 完整分析逻辑
        if (isProblematic(method)) {
            context.report(
                Incident(context, ISSUE)
                    .message("Problematic method call: ${method.name}")
                    .at(node)
            )
        }
    }
    
    companion object {
        private val KNOWN_PROBLEMATIC_METHODS = setOf(
            "dangerousMethod",
            "deprecatedCall"
        )
    }
}
```

### 部分分析策略

#### 1. 使用启发式方法

```kotlin
class HeuristicDetector : Detector(), SourceCodeScanner {
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 使用方法名和参数模式进行启发式检测
        val methodName = node.methodName ?: return
        val argCount = node.valueArgumentCount
        
        when {
            methodName == "findViewById" && argCount == 1 -> {
                checkFindViewById(context, node)
            }
            methodName.startsWith("get") && argCount == 0 -> {
                checkGetterMethod(context, node)
            }
            else -> {
                // 使用通用启发式规则
                checkGeneralPattern(context, node)
            }
        }
    }
    
    private fun checkFindViewById(context: JavaContext, node: UCallExpression) {
        // 即使没有完整的类型信息，也可以进行基本检查
        val arg = node.valueArguments.firstOrNull()
        if (arg is ULiteralExpression && arg.value is Int) {
            val resourceId = arg.value as Int
            if (resourceId == 0) {  // 常见错误模式
                context.report(
                    Incident(context, ISSUE)
                        .message("findViewById with resource ID 0")
                        .at(node)
                )
            }
        }
    }
}
```

#### 2. 缓存和重用信息

```kotlin
class CachingDetector : Detector(), SourceCodeScanner {
    
    private val methodCallCache = mutableMapOf<String, Boolean>()
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        val signature = getMethodSignature(node)
        
        // 使用缓存避免重复分析
        val isProblematic = methodCallCache.getOrPut(signature) {
            analyzeMethodCall(context, node, method)
        }
        
        if (isProblematic) {
            context.report(
                Incident(context, ISSUE)
                    .message("Cached problematic method: $signature")
                    .at(node)
            )
        }
    }
    
    private fun getMethodSignature(node: UCallExpression): String {
        val methodName = node.methodName ?: "unknown"
        val argTypes = node.valueArguments.joinToString(",") { 
            it.getExpressionType()?.canonicalText ?: "unknown"
        }
        return "$methodName($argTypes)"
    }
    
    private fun analyzeMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod): Boolean {
        // 执行实际的分析逻辑
        return isKnownProblematicPattern(node)
    }
}
```

### 处理依赖缺失

#### 优雅降级

```kotlin
class DependencyAwareDetector : Detector(), SourceCodeScanner {
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        val evaluator = context.evaluator
        
        // 尝试获取完整的类型信息
        val containingClass = method.containingClass
        
        if (containingClass != null) {
            // 完整信息可用
            performFullAnalysis(context, node, method, containingClass)
        } else {
            // 依赖缺失，使用降级策略
            performPartialAnalysis(context, node)
        }
    }
    
    private fun performFullAnalysis(
        context: JavaContext, 
        node: UCallExpression, 
        method: PsiMethod, 
        containingClass: PsiClass
    ) {
        // 使用完整的类型信息进行精确分析
        val className = containingClass.qualifiedName
        if (className in PROBLEMATIC_CLASSES) {
            context.report(
                Incident(context, ISSUE)
                    .message("Call to problematic class: $className")
                    .at(node)
            )
        }
    }
    
    private fun performPartialAnalysis(context: JavaContext, node: UCallExpression) {
        // 基于有限信息的分析
        val methodName = node.methodName
        val receiverText = node.receiver?.asSourceString()
        
        // 使用字符串模式匹配
        if (receiverText != null && methodName != null) {
            val pattern = "$receiverText.$methodName"
            if (PROBLEMATIC_PATTERNS.any { pattern.contains(it) }) {
                context.report(
                    Incident(context, ISSUE)
                        .message("Potentially problematic pattern: $pattern")
                        .at(node)
                )
            }
        }
    }
    
    companion object {
        private val PROBLEMATIC_CLASSES = setOf(
            "com.example.ProblematicClass",
            "com.example.DeprecatedClass"
        )
        
        private val PROBLEMATIC_PATTERNS = setOf(
            "dangerousMethod",
            "unsafeOperation"
        )
    }
}
```

### 增量分析支持

#### 文件变更感知

```kotlin
class IncrementalDetector : Detector(), SourceCodeScanner {
    
    override fun beforeCheckRootProject(context: Context) {
        super.beforeCheckRootProject(context)
        
        // 在部分分析模式下，只分析变更的文件
        if (context.driver.isPartialAnalysis) {
            initializeIncrementalState(context)
        }
    }
    
    private fun initializeIncrementalState(context: Context) {
        // 加载之前的分析状态
        val previousState = loadPreviousAnalysisState(context)
        
        // 确定哪些文件发生了变更
        val changedFiles = getChangedFiles(context)
        
        // 只分析变更的文件
        context.driver.requestAnalysisOf(changedFiles)
    }
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 检查当前文件是否在增量分析范围内
        if (shouldAnalyzeFile(context.file)) {
            performAnalysis(context, node, method)
        }
    }
    
    private fun shouldAnalyzeFile(file: File): Boolean {
        // 实现文件过滤逻辑
        return true // 简化实现
    }
}
```

### 性能优化

#### 延迟计算

```kotlin
class LazyDetector : Detector(), SourceCodeScanner {
    
    // 延迟初始化的昂贵资源
    private val expensiveResource by lazy {
        initializeExpensiveResource()
    }
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 只在需要时才初始化昂贵资源
        if (needsExpensiveAnalysis(node)) {
            val result = expensiveResource.analyze(node)
            if (result.isProblematic) {
                context.report(
                    Incident(context, ISSUE)
                        .message("Expensive analysis found issue")
                        .at(node)
                )
            }
        } else {
            // 使用快速检查
            performQuickCheck(context, node)
        }
    }
    
    private fun needsExpensiveAnalysis(node: UCallExpression): Boolean {
        // 决定是否需要昂贵的分析
        return node.methodName in COMPLEX_METHODS
    }
    
    private fun performQuickCheck(context: JavaContext, node: UCallExpression) {
        // 执行快速、简单的检查
        if (node.methodName in SIMPLE_PROBLEMATIC_METHODS) {
            context.report(
                Incident(context, ISSUE)
                    .message("Quick check found issue")
                    .at(node)
            )
        }
    }
    
    companion object {
        private val COMPLEX_METHODS = setOf("complexMethod", "heavyOperation")
        private val SIMPLE_PROBLEMATIC_METHODS = setOf("simpleError", "quickIssue")
    }
}
```

### 测试部分分析

#### 测试不完整信息

```kotlin
@Test
fun testPartialAnalysisWithMissingDependencies() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                
                class TestClass {
                    fun method() {
                        // 调用可能不存在的类
                        MissingClass.problematicMethod()
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .partial()
        .allowCompilationErrors()  // 允许编译错误
        .run()
        .expect("""
            src/test/pkg/TestClass.kt:5: Warning: Potentially problematic pattern [IssueId]
                    MissingClass.problematicMethod()
                    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
            0 errors, 1 warnings
        """.trimIndent())
}
```

#### 测试增量模式

```kotlin
@Test
fun testIncrementalAnalysis() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        problematicCall()
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .incremental("src/test/pkg/TestClass.kt")  // 只分析指定文件
        .run()
        .expect("""
            src/test/pkg/TestClass.kt:4: Warning: Issue detected [IssueId]
                    problematicCall()
                    ~~~~~~~~~~~~~~~~~
            0 errors, 1 warnings
        """.trimIndent())
}
```

### 最佳实践

#### 1. 设计可降级的检查

```kotlin
// 好的做法：可以在部分信息下工作
class GoodDetector : Detector(), SourceCodeScanner {
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        val methodName = node.methodName
        if (methodName in KNOWN_ISSUES) {
            // 基于方法名的检查，不依赖完整的类型信息
            reportIssue(context, node)
        }
    }
}

// 不好的做法：完全依赖类型解析
class BadDetector : Detector(), SourceCodeScanner {
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        val containingClass = method.containingClass ?: return  // 在部分分析中可能为null
        val className = containingClass.qualifiedName ?: return
        // 这种检查在部分分析中可能失效
    }
}
```

#### 2. 提供配置选项

```kotlin
class ConfigurableDetector : Detector(), SourceCodeScanner {
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        val isPartialAnalysis = context.driver.isPartialAnalysis
        val strictMode = getOption(STRICT_MODE_OPTION, context) ?: !isPartialAnalysis
        
        if (strictMode) {
            performStrictAnalysis(context, node, method)
        } else {
            performLenientAnalysis(context, node)
        }
    }
    
    companion object {
        private val STRICT_MODE_OPTION = BooleanOption(
            "strict-mode",
            "Enable strict analysis mode",
            false
        )
    }
}
```

#### 3. 文档化部分分析行为

```kotlin
class DocumentedDetector : Detector(), SourceCodeScanner {
    
    companion object {
        val ISSUE = Issue.create(
            id = "DocumentedIssue",
            briefDescription = "Issue with documented partial analysis behavior",
            explanation = """
                This check detects problematic patterns in code.
                
                **Partial Analysis**: In partial analysis mode, this check uses 
                heuristic methods and may produce different results than full analysis.
                It relies on method names and basic patterns rather than complete
                type information.
                """,
            category = Category.CORRECTNESS,
            severity = Severity.WARNING,
            implementation = Implementation(
                DocumentedDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```
