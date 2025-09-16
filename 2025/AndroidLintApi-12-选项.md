# Android Lint API 指南 - 选项

## 选项

Lint提供了多种配置选项来控制检查的行为。这些选项可以通过多种方式设置，包括配置文件、命令行参数和代码注解。

### lint.xml配置文件

`lint.xml`是主要的Lint配置文件，通常放在项目根目录中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- 禁用特定检查 -->
    <issue id="IconMissingDensityFolder" severity="ignore" />
    
    <!-- 将警告提升为错误 -->
    <issue id="HardcodedText" severity="error" />
    
    <!-- 为特定路径禁用检查 -->
    <issue id="UnusedResources">
        <ignore path="src/test/*" />
        <ignore path="*/generated/*" />
    </issue>
    
    <!-- 使用正则表达式匹配路径 -->
    <issue id="NewApi">
        <ignore regexp=".*/(test|androidTest)/.*" />
    </issue>
    
    <!-- 配置检查的严重性级别 -->
    <issue id="MissingTranslation" severity="warning">
        <!-- 忽略特定语言 -->
        <ignore path="res/values-en/*" />
    </issue>
</lint>
```

### 严重性级别

Lint支持以下严重性级别：

- `fatal`：致命错误，会中断构建
- `error`：错误，通常会中断构建
- `warning`：警告，不会中断构建但应该修复
- `informational`：信息性消息
- `ignore`：忽略此问题

```xml
<lint>
    <issue id="CustomIssue" severity="error" />
    <issue id="PerformanceIssue" severity="warning" />
    <issue id="StyleIssue" severity="informational" />
    <issue id="LegacyIssue" severity="ignore" />
</lint>
```

### Gradle配置

在`build.gradle`文件中配置Lint选项：

```gradle
android {
    lintOptions {
        // 遇到错误时不中断构建
        abortOnError false
        
        // 检查所有问题，包括默认禁用的
        checkAllWarnings true
        
        // 将所有警告视为错误
        warningsAsErrors true
        
        // 禁用特定检查
        disable 'TypographyFractions', 'TypographyQuotes'
        
        // 启用特定检查
        enable 'RtlHardcoded', 'RtlCompat', 'RtlEnabled'
        
        // 只检查这些问题
        check 'NewApi', 'InlinedApi'
        
        // 设置基线文件
        baseline file("lint-baseline.xml")
        
        // 输出选项
        htmlReport true
        htmlOutput file("$project.buildDir/reports/lint/lint.html")
        xmlReport true
        xmlOutput file("$project.buildDir/reports/lint/lint.xml")
        
        // 检查依赖项
        checkDependencies true
        
        // 检查生成的源代码
        checkGeneratedSources true
        
        // 检查测试源代码
        checkTestSources true
        
        // 忽略测试源代码
        ignoreTestSources true
        
        // 忽略指定的变体
        ignore 'InvalidPackage'
        
        // 设置问题的严重性
        error 'Wakelock', 'TextViewEdits'
        warning 'ResourceAsColor'
        informational 'StopShip'
        
        // 致命问题
        fatal 'NewApi', 'InlineApi'
        
        // 设置文本输出
        textReport true
        textOutput 'stdout'
        
        // 检查发布构建
        checkReleaseBuilds false
        
        // 静默模式
        quiet true
        
        // 详细输出
        verbose true
    }
}
```

### 命令行选项

直接使用Lint命令行工具时的选项：

```bash
# 基本用法
lint [flags] <project directory>

# 常用选项
lint --help                          # 显示帮助
lint --list                          # 列出所有可用的问题ID
lint --show UnusedResources          # 显示特定问题的详细信息

# 输出控制
lint --html output.html              # HTML输出
lint --xml output.xml                # XML输出  
lint --text output.txt               # 文本输出
lint --sarif output.sarif            # SARIF输出

# 检查控制
lint --check NewApi                  # 只检查特定问题
lint --disable TypographyFractions   # 禁用特定检查
lint --enable RtlHardcoded           # 启用特定检查
lint --ignore InvalidPackage        # 忽略特定问题

# 严重性控制
lint --error Wakelock               # 将问题视为错误
lint --warning ResourceAsColor      # 将问题视为警告
lint --fatal NewApi                 # 将问题视为致命错误

# 其他选项
lint --config lint.xml              # 指定配置文件
lint --baseline lint-baseline.xml   # 指定基线文件
lint --update-baseline              # 更新基线文件
lint --abort-if-errors-found        # 发现错误时退出
lint --quiet                        # 静默模式
lint --verbose                      # 详细输出
lint --check-dependencies           # 检查依赖项
lint --check-generated-sources      # 检查生成的源代码
```

### 基线文件

基线文件允许您记录当前的问题状态，只关注新引入的问题：

```xml
<!-- lint-baseline.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<issues format="6" by="lint 7.0.0">
    
    <issue
        id="HardcodedText"
        message="Hardcoded string &quot;Hello&quot;, should use `@string` resource"
        errorLine1="        android:text=&quot;Hello&quot; />"
        errorLine2="        ~~~~~~~~~~~~~~~~">
        <location
            file="src/main/res/layout/activity_main.xml"
            line="12"
            column="9"/>
    </issue>
    
</issues>
```

生成和使用基线文件：

```bash
# 生成基线文件
lint --baseline lint-baseline.xml .

# 使用基线文件
lint --baseline lint-baseline.xml .

# 更新基线文件
lint --update-baseline --baseline lint-baseline.xml .
```

### 自定义检查选项

为自定义Lint检查添加配置选项：

```kotlin
class ConfigurableDetector : Detector(), Detector.UastScanner {
    
    companion object {
        // 定义配置选项
        val MAX_LINES_OPTION = IntOption(
            "maxLines",
            "方法的最大行数",
            50,  // 默认值
            "方法超过此行数时报告警告"
        )
        
        val ALLOWED_EXCEPTIONS_OPTION = StringOption(
            "allowedExceptions",
            "允许的异常类型",
            "java.lang.RuntimeException,java.lang.IllegalArgumentException",
            "逗号分隔的允许抛出的异常类型列表"
        )
        
        val ISSUE = Issue.create(
            id = "MethodTooLong",
            briefDescription = "方法过长",
            explanation = "方法不应该超过配置的最大行数",
            category = Category.PERFORMANCE,
            priority = 5,
            severity = Severity.WARNING,
            implementation = Implementation(
                ConfigurableDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            ),
            options = listOf(MAX_LINES_OPTION, ALLOWED_EXCEPTIONS_OPTION)
        )
    }
    
    override fun visitMethod(context: JavaContext, node: UMethod) {
        val maxLines = MAX_LINES_OPTION.getValue(context)
        val allowedExceptions = ALLOWED_EXCEPTIONS_OPTION.getValue(context)
            .split(",")
            .map { it.trim() }
        
        val lineCount = getMethodLineCount(node)
        
        if (lineCount > maxLines && !hasAllowedException(node, allowedExceptions)) {
            context.report(
                ISSUE,
                node,
                context.getNameLocation(node),
                "方法有${lineCount}行，超过了最大限制${maxLines}行"
            )
        }
    }
    
    private fun getMethodLineCount(method: UMethod): Int {
        val body = method.uastBody ?: return 0
        val startLine = body.sourcePsi?.textRange?.startOffset ?: return 0
        val endLine = body.sourcePsi?.textRange?.endOffset ?: return 0
        // 计算行数的逻辑
        return endLine - startLine
    }
    
    private fun hasAllowedException(method: UMethod, allowedExceptions: List<String>): Boolean {
        // 检查方法是否抛出允许的异常
        return method.throwsList.any { exception ->
            allowedExceptions.contains(exception.qualifiedName)
        }
    }
}
```

### 配置选项类型

Lint支持多种配置选项类型：

```kotlin
// 整数选项
val INT_OPTION = IntOption(
    "intValue",
    "整数配置",
    42,
    "这是一个整数配置选项"
)

// 字符串选项
val STRING_OPTION = StringOption(
    "stringValue", 
    "字符串配置",
    "default",
    "这是一个字符串配置选项"
)

// 布尔选项
val BOOLEAN_OPTION = BooleanOption(
    "booleanValue",
    "布尔配置", 
    true,
    "这是一个布尔配置选项"
)

// 文件选项
val FILE_OPTION = FileOption(
    "configFile",
    "配置文件",
    null,
    "指定配置文件的路径"
)
```

### 项目特定配置

不同模块可以有不同的Lint配置：

```
project/
├── lint.xml                 # 全局配置
├── app/
│   ├── lint.xml            # app模块配置
│   └── build.gradle
├── library/
│   ├── lint.xml            # library模块配置
│   └── build.gradle
└── build.gradle
```

模块特定的配置会覆盖全局配置：

```xml
<!-- app/lint.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- 在app模块中允许硬编码文本 -->
    <issue id="HardcodedText" severity="ignore" />
</lint>
```

### 环境变量

某些Lint行为可以通过环境变量控制：

```bash
# 设置Lint缓存目录
export LINT_CACHE_DIR=/tmp/lint-cache

# 禁用Lint缓存
export LINT_DISABLE_CACHE=true

# 设置最大内存使用
export LINT_MAX_HEAP_SIZE=2g

# 启用调试输出
export LINT_DEBUG=true
```

### 最佳实践

1. **渐进式改进**：使用基线文件逐步改善代码质量
2. **团队一致性**：确保团队使用相同的Lint配置
3. **CI/CD集成**：在持续集成中运行Lint检查
4. **定期审查**：定期审查和更新Lint配置
5. **文档化配置**：为自定义配置添加注释和文档

```xml
<!-- 良好的配置示例 -->
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- 性能相关检查设为错误 -->
    <issue id="Wakelock" severity="error" />
    <issue id="Recycle" severity="error" />
    
    <!-- UI相关检查设为警告 -->
    <issue id="HardcodedText" severity="warning" />
    <issue id="ContentDescription" severity="warning" />
    
    <!-- 忽略第三方库的问题 -->
    <issue id="InvalidPackage">
        <ignore path="**/third_party/**" />
    </issue>
    
    <!-- 测试代码的特殊规则 -->
    <issue id="SetTextI18n">
        <ignore path="src/test/**" />
        <ignore path="src/androidTest/**" />
    </issue>
</lint>
```

通过合理配置Lint选项，您可以创建适合项目需求的代码质量检查体系，提高代码质量和开发效率。 