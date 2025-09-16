# Android Lint API 指南 - 常见问题

## 常见问题

本章收集了在开发Android Lint检查时经常遇到的问题和解决方案。

### 一般问题

#### Q: 如何调试Lint检查？

**A:** 有几种调试Lint检查的方法：

1. **使用IDE调试**：

```kotlin
// 在检查代码中设置断点
override fun visitMethodCall(
    context: JavaContext,
    node: UCallExpression,
    method: PsiMethod
) {
    // 设置断点在这里
    val methodName = method.name
    if (methodName == "targetMethod") {
        // 调试逻辑
    }
}
```

2. **使用日志输出**：

```kotlin
override fun visitMethodCall(
    context: JavaContext,
    node: UCallExpression,
    method: PsiMethod
) {
    System.err.println("Visiting method: ${method.name}")
    System.err.println("In class: ${method.containingClass?.qualifiedName}")
}
```

3. **使用测试驱动开发**：

```kotlin
@Test
fun testDebugScenario() {
    lint()
        .files(
            java("""
                public class Test {
                    public void problematicMethod() {
                        // 测试场景
                    }
                }
            """).indented()
        )
        .run()
        .expectClean() // 或 .expect("expected output")
}
```

#### Q: 为什么我的检查没有运行？

**A:** 常见原因包括：

1. **作用域不正确**：

```kotlin
// 错误：使用了错误的作用域
implementation = Implementation(
    MyDetector::class.java,
    Scope.RESOURCE_FILE_SCOPE // 但检查的是Java代码
)

// 正确：使用正确的作用域
implementation = Implementation(
    MyDetector::class.java,
    Scope.JAVA_FILE_SCOPE
)
```

2. **扫描器接口缺失**：

```kotlin
// 错误：没有实现相应的扫描器接口
class MyDetector : Detector()

// 正确：实现了相应的接口
class MyDetector : Detector(), Detector.UastScanner
```

3. **Issue注册问题**：

```kotlin
// 确保在IssueRegistry中注册了Issue
class MyIssueRegistry : IssueRegistry() {
    override val issues: List<Issue>
        get() = listOf(MyDetector.ISSUE) // 确保包含了所有Issue
}
```

#### Q: 如何处理不同的Android API级别？

**A:** 使用context提供的API信息：

```kotlin
override fun visitMethodCall(
    context: JavaContext,
    node: UCallExpression,
    method: PsiMethod
) {
    val minSdk = context.mainProject.minSdk
    val targetSdk = context.mainProject.targetSdk
    val compileSdk = context.mainProject.compileSdk
    
    // 检查API级别
    if (minSdk < 21) {
        // 处理低API级别的情况
    }
}
```

### UAST相关问题

#### Q: 如何区分Java和Kotlin代码？

**A:** 检查源文件的语言：

```kotlin
override fun visitClass(context: JavaContext, node: UClass) {
    val isKotlin = node.lang == UastLanguagePlugin.KOTLIN_LANGUAGE
    val isJava = node.lang == UastLanguagePlugin.JAVA_LANGUAGE
    
    if (isKotlin) {
        // Kotlin特定的处理
    } else if (isJava) {
        // Java特定的处理
    }
}
```

#### Q: 如何获取方法的完整签名？

**A:** 使用JavaEvaluator：

```kotlin
override fun visitMethodCall(
    context: JavaContext,
    node: UCallExpression,
    method: PsiMethod
) {
    val evaluator = context.evaluator
    val signature = evaluator.getMethodDescription(
        method,
        includeArgumentNames = false,
        includeArgumentTypes = true
    )
    // 例如: "void setTitle(java.lang.String)"
}
```

#### Q: 如何检查继承关系？

**A:** 使用evaluator的继承检查方法：

```kotlin
override fun visitClass(context: JavaContext, node: UClass) {
    val evaluator = context.evaluator
    
    // 检查是否继承自特定类
    if (evaluator.extendsClass(node.javaPsi, "android.app.Activity", false)) {
        // 这是一个Activity
    }
    
    // 检查是否实现了特定接口
    if (evaluator.implementsInterface(node.javaPsi, "java.io.Serializable", false)) {
        // 实现了Serializable接口
    }
}
```

### 性能问题

#### Q: 如何优化Lint检查的性能？

**A:** 几个优化策略：

1. **限制检查范围**：

```kotlin
// 只检查特定的方法名
override fun getApplicableMethodNames(): List<String> {
    return listOf("findViewById", "getSystemService")
}

// 而不是检查所有方法调用
override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
    if (method.name in listOf("findViewById", "getSystemService")) {
        // 处理
    }
}
```

2. **缓存昂贵的计算**：

```kotlin
class OptimizedDetector : Detector(), Detector.UastScanner {
    private val cache = mutableMapOf<String, Boolean>()
    
    private fun isExpensiveCheck(className: String): Boolean {
        return cache.getOrPut(className) {
            // 昂贵的计算
            performExpensiveAnalysis(className)
        }
    }
}
```

3. **避免深度递归**：

```kotlin
private fun analyzeWithDepthLimit(node: UElement, depth: Int = 0) {
    if (depth > MAX_DEPTH) return
    
    node.accept(object : AbstractUastVisitor() {
        override fun visitElement(node: UElement): Boolean {
            analyzeWithDepthLimit(node, depth + 1)
            return false
        }
    })
}
```

#### Q: 为什么Lint检查很慢？

**A:** 常见的性能问题：

1. **过度使用字符串操作**：

```kotlin
// 慢
fun isTargetMethod(method: PsiMethod): Boolean {
    return method.containingClass?.qualifiedName?.contains("MyClass") == true
}

// 快
fun isTargetMethod(method: PsiMethod): Boolean {
    return method.containingClass?.qualifiedName == "com.example.MyClass"
}
```

2. **重复的PSI遍历**：

```kotlin
// 慢：多次遍历
override fun visitMethod(context: JavaContext, node: UMethod) {
    val calls = findMethodCalls(node) // 遍历1
    val variables = findVariables(node) // 遍历2
}

// 快：单次遍历
override fun visitMethod(context: JavaContext, node: UMethod) {
    val calls = mutableListOf<UCallExpression>()
    val variables = mutableListOf<UVariable>()
    
    node.accept(object : AbstractUastVisitor() {
        override fun visitCallExpression(node: UCallExpression): Boolean {
            calls.add(node)
            return super.visitCallExpression(node)
        }
        
        override fun visitVariable(node: UVariable): Boolean {
            variables.add(node)
            return super.visitVariable(node)
        }
    })
}
```

### 测试问题

#### Q: 如何测试复杂的场景？

**A:** 使用多文件测试：

```kotlin
@Test
fun testComplexScenario() {
    lint()
        .files(
            java("""
                public class MainActivity extends Activity {
                    @Override
                    protected void onCreate(Bundle savedInstanceState) {
                        // 测试代码
                    }
                }
            """).indented(),
            xml("src/main/AndroidManifest.xml", """
                <manifest xmlns:android="http://schemas.android.com/apk/res/android">
                    <application>
                        <activity android:name=".MainActivity" />
                    </application>
                </manifest>
            """).indented(),
            gradle("""
                android {
                    compileSdkVersion 30
                    defaultConfig {
                        minSdkVersion 21
                        targetSdkVersion 30
                    }
                }
            """).indented()
        )
        .run()
        .expect("expected output")
}
```

#### Q: 如何测试快速修复？

**A:** 使用checkFix方法：

```kotlin
@Test
fun testQuickFix() {
    lint()
        .files(
            java("""
                public class Test {
                    public void method() {
                        findViewById(R.id.button); // 问题代码
                    }
                }
            """).indented()
        )
        .run()
        .expect("""
            src/Test.java:3: Warning: Use view binding instead [UseViewBinding]
                findViewById(R.id.button);
                ~~~~~~~~~~~~~~~~~~~~~~~~~~
            0 errors, 1 warnings
        """.trimIndent())
        .checkFix(
            null, // 使用默认修复
            java("""
                public class Test {
                    public void method() {
                        binding.button; // 修复后的代码
                    }
                }
            """).indented()
        )
}
```

### 配置问题

#### Q: 如何让用户配置检查行为？

**A:** 使用Option系统：

```kotlin
companion object {
    val MAX_PARAMETERS = IntOption(
        "maxParameters",
        "方法的最大参数数量",
        5,
        "超过此数量的参数会触发警告"
    )
    
    val ISSUE = Issue.create(
        id = "TooManyParameters",
        briefDescription = "方法参数过多",
        explanation = "方法不应该有太多参数",
        category = Category.CORRECTNESS,
        priority = 5,
        severity = Severity.WARNING,
        implementation = Implementation(
            ParameterCountDetector::class.java,
            Scope.JAVA_FILE_SCOPE
        ),
        options = listOf(MAX_PARAMETERS)
    )
}

override fun visitMethod(context: JavaContext, node: UMethod) {
    val maxParams = MAX_PARAMETERS.getValue(context)
    if (node.uastParameters.size > maxParams) {
        // 报告问题
    }
}
```

#### Q: 如何处理不同的构建变体？

**A:** 检查构建变体信息：

```kotlin
override fun visitClass(context: JavaContext, node: UClass) {
    val variant = context.project.buildVariant
    val isDebug = variant?.buildType == "debug"
    val isRelease = variant?.buildType == "release"
    
    if (isDebug) {
        // 调试版本的特殊处理
    } else if (isRelease) {
        // 发布版本的特殊处理
    }
}
```

### 常见错误

#### Q: 为什么会出现ClassCastException？

**A:** 通常是因为类型转换错误：

```kotlin
// 错误：直接转换
val method = node.resolve() as PsiMethod

// 正确：安全转换
val method = node.resolve() as? PsiMethod ?: return
```

#### Q: 为什么会出现NullPointerException？

**A:** 总是检查null值：

```kotlin
// 错误：没有检查null
val className = method.containingClass.qualifiedName

// 正确：检查null
val className = method.containingClass?.qualifiedName ?: return
```

#### Q: 如何处理生成的代码？

**A:** 检查文件是否是生成的：

```kotlin
override fun visitClass(context: JavaContext, node: UClass) {
    if (context.file.name.contains("generated") || 
        context.file.path.contains("/generated/")) {
        return // 跳过生成的代码
    }
    
    // 或者检查注解
    if (context.evaluator.findAnnotation(node.javaPsi, "javax.annotation.Generated") != null) {
        return
    }
}
```

### 部署问题

#### Q: 如何分发自定义Lint检查？

**A:** 几种分发方式：

1. **作为AAR库**：

```gradle
// library/build.gradle
android {
    lintOptions {
        checkDependencies true
    }
}

dependencies {
    lintPublish project(':lint-checks')
}
```

2. **作为独立JAR**：

```gradle
// app/build.gradle
dependencies {
    lintChecks files('libs/my-lint-checks.jar')
}
```

3. **通过Maven/Gradle发布**：

```gradle
publishing {
    publications {
        maven(MavenPublication) {
            artifact lintJar
        }
    }
}
```

#### Q: 如何确保团队使用相同的Lint配置？

**A:** 使用版本控制管理配置：

```
project/
├── lint.xml              # 提交到版本控制
├── lint-baseline.xml     # 提交到版本控制
└── gradle.properties     # 包含Lint配置
```

```gradle
// gradle.properties
lint.abortOnError=true
lint.checkAllWarnings=true
```

### 最佳实践

1. **始终编写测试**：确保检查按预期工作
2. **处理边界情况**：考虑null值、空集合等
3. **优化性能**：避免不必要的计算
4. **提供清晰的错误消息**：帮助开发者理解和修复问题
5. **文档化配置选项**：让用户知道如何自定义行为

通过了解这些常见问题和解决方案，您可以更有效地开发和维护Android Lint检查。 