# Android Lint API 指南 - 第8章：添加快速修复


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第8章节"Adding Quick Fixes"的中文翻译版本。

## 添加快速修复

快速修复（Quick Fix）允许IDE自动修复lint检查发现的问题。这大大提高了开发者的效率，因为他们不需要手动修复每个问题。

### 基本快速修复

#### 创建简单的快速修复

最简单的快速修复是文本替换：

```kotlin
class MyDetector : Detector(), SourceCodeScanner {
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        if (isProblematicCall(node, method)) {
            val fix = fix()
                .name("Replace with correct method")
                .replace()
                .text("problematicMethod()")
                .with("correctMethod()")
                .build()
            
            context.report(
                Incident(context, ISSUE)
                    .message("Use correctMethod() instead")
                    .at(node)
                    .fix(fix)
            )
        }
    }
}
```

#### 报告时添加快速修复

```kotlin
// 使用新的Incident API (Lint 7.0+)
context.report(
    Incident(context, ISSUE)
        .message("问题描述")
        .at(location)
        .fix(quickFix)
)

// 使用旧的API (仍然支持)
context.report(
    ISSUE, 
    location, 
    "问题描述", 
    quickFix
)
```

### 修复类型

#### 1. 文本替换

```kotlin
val fix = fix()
    .name("Replace deprecated method")
    .replace()
    .text("oldMethod()")
    .with("newMethod()")
    .build()
```

#### 2. 范围替换

```kotlin
val fix = fix()
    .name("Fix method call")
    .replace()
    .range(context.getLocation(node))
    .with("newCode()")
    .build()
```

#### 3. 正则表达式替换

```kotlin
val fix = fix()
    .name("Fix pattern")
    .replace()
    .pattern("old(\\w+)")
    .with("new$1")
    .build()
```

#### 4. 删除内容

```kotlin
val fix = fix()
    .name("Remove unused import")
    .replace()
    .range(context.getLocation(importStatement))
    .with("")
    .build()
```

### 高级快速修复

#### 多步修复

```kotlin
val fix = fix()
    .name("Complete refactor")
    .composite(
        // 第一步：替换方法调用
        fix()
            .replace()
            .text("oldMethod()")
            .with("newMethod()")
            .build(),
        // 第二步：添加导入
        fix()
            .replace()
            .beginning()
            .with("import com.example.NewClass;\n")
            .build()
    )
```

#### 条件修复

```kotlin
val fixes = mutableListOf<LintFix>()

// 添加基本修复
fixes.add(
    fix()
        .name("Replace with recommended method")
        .replace()
        .text("problematicCall()")
        .with("recommendedCall()")
        .build()
)

// 如果有替代方案，添加额外选项
if (hasAlternative) {
    fixes.add(
        fix()
            .name("Use alternative approach")
            .replace()
            .text("problematicCall()")
            .with("alternativeCall()")
            .build()
    )
}

// 创建选择修复
val compositeFix = fix()
    .alternatives(*fixes.toTypedArray())

context.report(
    Incident(context, ISSUE)
        .message("Multiple solutions available")
        .at(location)
        .fix(compositeFix)
)
```

### 智能快速修复

#### 基于上下文的修复

```kotlin
override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
    if (needsNullCheck(node)) {
        val variableName = getVariableName(node)
        val fix = when {
            isInKotlin(context) -> createKotlinNullCheckFix(variableName)
            else -> createJavaNullCheckFix(variableName)
        }
        
        context.report(
            Incident(context, ISSUE)
                .message("Add null check")
                .at(node)
                .fix(fix)
        )
    }
}

private fun createKotlinNullCheckFix(variableName: String): LintFix {
    return fix()
        .name("Add null check (Kotlin)")
        .replace()
        .text("$variableName.method()")
        .with("$variableName?.method()")
        .build()
}

private fun createJavaNullCheckFix(variableName: String): LintFix {
    return fix()
        .name("Add null check (Java)")
        .replace()
        .text("$variableName.method()")
        .with("if ($variableName != null) $variableName.method()")
        .build()
}
```

#### 模板化修复

```kotlin
private fun createGenericFix(oldPattern: String, newTemplate: String, vararg args: String): LintFix {
    val replacement = String.format(newTemplate, *args)
    return fix()
        .name("Apply template fix")
        .replace()
        .text(oldPattern)
        .with(replacement)
        .build()
}

// 使用示例
val fix = createGenericFix(
    "Collections.singletonList(%s)",
    "listOf(%s)",
    argument
)
```

### 位置精确修复

#### 使用精确位置

```kotlin
override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
    // 获取方法名的精确位置
    val methodNameLocation = context.getNameLocation(node)
    
    val fix = fix()
        .name("Rename method")
        .replace()
        .range(methodNameLocation)
        .with("newMethodName")
        .build()
    
    context.report(
        Incident(context, ISSUE)
            .message("Method should be renamed")
            .at(methodNameLocation)
            .fix(fix)
    )
}
```

#### 多位置修复

```kotlin
private fun createMultiLocationFix(context: JavaContext, calls: List<UCallExpression>): LintFix {
    val fixes = calls.map { call ->
        fix()
            .replace()
            .range(context.getLocation(call))
            .with("newMethod()")
            .build()
    }
    
    return fix()
        .name("Replace all occurrences")
        .composite(*fixes.toTypedArray())
}
```

### 文件操作修复

#### 添加导入

```kotlin
private fun createImportFix(context: JavaContext, importStatement: String): LintFix {
    return fix()
        .name("Add missing import")
        .replace()
        .beginning()  // 文件开头
        .with("$importStatement\n")
        .build()
}
```

#### 添加注解

```kotlin
private fun createAnnotationFix(context: JavaContext, node: UMethod): LintFix {
    val location = context.getLocation(node)
    return fix()
        .name("Add @Override annotation")
        .replace()
        .range(location.start, location.start)  // 在方法开始处插入
        .with("@Override\n    ")
        .build()
}
```

#### 创建新文件

```kotlin
private fun createNewFileFix(context: Context, fileName: String, content: String): LintFix {
    return fix()
        .name("Create $fileName")
        .newFile(File(context.project.dir, fileName), content)
        .build()
}
```

### 交互式快速修复

#### 带用户输入的修复

```kotlin
private fun createInteractiveFix(): LintFix {
    return fix()
        .name("Rename variable")
        .replace()
        .text("oldName")
        .with("TODO")  // 用户需要替换的占位符
        .select("TODO")  // 选中占位符文本
        .build()
}
```

#### 选择列表修复

```kotlin
private fun createChoiceFix(options: List<String>): LintFix {
    val fixes = options.map { option ->
        fix()
            .name("Use $option")
            .replace()
            .text("placeholder")
            .with(option)
            .build()
    }
    
    return fix()
        .alternatives(*fixes.toTypedArray())
}
```

### 快速修复测试

#### 基本修复测试

```kotlin
@Test
fun testQuickFix() {
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
        .run()
        .expect("""
            src/test/pkg/TestClass.kt:4: Warning: Use recommended method [IssueId]
                    problematicCall()
                    ~~~~~~~~~~~~~~~~~
            0 errors, 1 warnings
        """.trimIndent())
        .expectFixDiffs("""
            Fix for src/test/pkg/TestClass.kt line 4: Replace with recommended method:
            @@ -4 +4
            -         problematicCall()
            +         recommendedCall()
        """.trimIndent())
}
```

#### 测试多个修复选项

```kotlin
@Test
fun testMultipleFixes() {
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
        .run()
        .expectFixDiffs("""
            Fix for src/test/pkg/TestClass.kt line 4: Replace with recommended method:
            @@ -4 +4
            -         problematicCall()
            +         recommendedCall()
            Fix for src/test/pkg/TestClass.kt line 4: Use alternative approach:
            @@ -4 +4
            -         problematicCall()
            +         alternativeCall()
        """.trimIndent())
}
```

#### 测试复合修复

```kotlin
@Test
fun testCompositeFix() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        oldMethod()
                        anotherOldMethod()
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expectFixDiffs("""
            Fix for src/test/pkg/TestClass.kt line 4: Complete refactor:
            @@ -4 +4
            -         oldMethod()
            +         newMethod()
            @@ -5 +5
            -         anotherOldMethod()
            +         anotherNewMethod()
        """.trimIndent())
}
```

### 快速修复最佳实践

#### 1. 提供清晰的修复名称

```kotlin
// 好的做法
fix().name("Replace deprecated getString() with getStringOrNull()")

// 不好的做法
fix().name("Fix")
```

#### 2. 验证修复的正确性

```kotlin
private fun createSafeFix(context: JavaContext, node: UCallExpression): LintFix? {
    // 验证修复是否适用
    if (!isFixApplicable(node)) {
        return null
    }
    
    return fix()
        .name("Safe replacement")
        .replace()
        .range(context.getLocation(node))
        .with(getSafeReplacement(node))
        .build()
}
```

#### 3. 处理边缘情况

```kotlin
private fun createRobustFix(context: JavaContext, node: UCallExpression): LintFix {
    val replacement = when {
        isInTryCatch(node) -> "safeMethod()"
        hasNullableReceiver(node) -> "nullSafeMethod()"
        else -> "defaultMethod()"
    }
    
    return fix()
        .name("Context-aware replacement")
        .replace()
        .range(context.getLocation(node))
        .with(replacement)
        .build()
}
```

#### 4. 提供多种修复选项

```kotlin
private fun createFlexibleFix(context: JavaContext, node: UCallExpression): LintFix {
    return fix()
        .alternatives(
            // 保守修复
            fix()
                .name("Safe fix (recommended)")
                .replace()
                .range(context.getLocation(node))
                .with("safeReplacement()")
                .build(),
            // 激进修复
            fix()
                .name("Complete refactor")
                .replace()
                .range(context.getLocation(node))
                .with("modernReplacement()")
                .build(),
            // 抑制警告
            fix()
                .name("Suppress warning")
                .replace()
                .range(context.getLocation(node.uastParent))
                .with("@Suppress(\"IssueId\")\n${node.uastParent?.asSourceString()}")
                .build()
        )
}
```

### 性能考虑

#### 延迟创建修复

```kotlin
override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
    if (isProblematic(node)) {
        // 延迟创建修复，只有在需要时才创建
        val fixSupplier = { createExpensiveFix(context, node) }
        
        context.report(
            Incident(context, ISSUE)
                .message("Problem detected")
                .at(node)
                .fix(fixSupplier)  // 传递供应商而不是实际修复
        )
    }
}
```

#### 缓存修复模板

```kotlin
companion object {
    private val FIX_TEMPLATES = mapOf(
        "deprecated_method" to "Use %s instead of %s",
        "null_check" to "Add null check for %s"
    )
}

private fun getCachedFix(template: String, vararg args: String): LintFix {
    val pattern = FIX_TEMPLATES[template] ?: return createDefaultFix()
    return fix()
        .name(String.format(pattern, *args))
        .replace()
        .text(args[1])
        .with(args[0])
        .build()
}
```
