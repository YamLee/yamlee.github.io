# Android Lint API 指南 - 第3章：示例GitHub示例Lint检查项目


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第3章节"Example: Sample Lint Check GitHub Project"的中文翻译版本。

## 示例：GitHub示例Lint检查项目

[https://github.com/googlesamples/android-custom-lint-rules](https://github.com/googlesamples/android-custom-lint-rules) GitHub项目提供了一个示例lint检查，展示了一个工作的骨架。

本章节将通过该示例项目并解释其内容和原因。

### 项目布局

这是示例项目的项目布局：

```
implementationlintPublish:app:library:checks
```

我们有一个应用程序模块`app`，它依赖（通过`implementation`依赖）于`library`，库本身对`checks`项目有`lintPublish`依赖。

### :checks

`checks`项目是实际实现lint检查的地方。这个项目是一个纯Kotlin或纯Java Gradle项目：

```groovy
apply plugin: 'java-library'
apply plugin: 'kotlin'
```

如果你查看示例项目，你会看到应用了第三个插件：`apply plugin: 'com.android.lint'`。这引入了独立的Lint Gradle插件，它向这个Kotlin项目添加了一个lint目标。这意味着你可以在`:checks`项目上运行`./gradlew lint`。这很有用，因为lint附带了十几个lint检查，用于查找lint检测器中的错误！这包括关于使用错误UAST方法、无效id格式、消息中看起来像代码的单词（应该用撇号包围）等的警告。

Gradle文件还声明了我们的检测器需要的lint API的依赖：

```groovy
dependencies {
    compileOnly "com.android.tools.lint:lint-api:$lintVersion"
    compileOnly "com.android.tools.lint:lint-checks:$lintVersion"
    testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
}
```

第二个依赖通常不是必需的；你只需要依赖Lint API。然而，内置检查定义了很多额外的基础设施，有时依赖它很方便，如`ApiLookup`，它让你查找给定方法所需的API级别等。在需要之前不要添加依赖。

### lintVersion？

上面定义的`lintVersion`变量是什么？

这是顶级build.gradle：

```groovy
buildscript {
    ext {
        kotlinVersion = '1.4.32'

        // Current lint target: Studio 4.2 / AGP 7
        //gradlePluginVersion = '4.2.0-beta06'
        //lintVersion = '27.2.0-beta06'

        // Upcoming lint target: Arctic Fox / AGP 7
        gradlePluginVersion = '7.0.0-alpha10'
        lintVersion = '30.0.0-alpha10'
    }

    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:$gradlePluginVersion"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion"
    }
}
```

`$lintVersion`变量在第11行定义。我们技术上不需要在这里定义`$gradlePluginVersion`或在第19行将其添加到类路径，但这样做是为了我们可以在检查本身上添加`lint`插件，以及其他模块`:app`和`:library`，它们确实需要它。

当你构建lint检查时，你正在针对在maven.google.com上分发的Lint API进行编译（在Gradle文件中通过`google()`引用）。这些遵循Gradle插件版本号。

因此，你首先选择你想要编译的lint API。如果可能，你应该使用最新可用的。

一旦你知道Gradle插件版本号，比如4.2.0-beta06，你可以通过简单地将**23**添加到gradle插件的主版本号来计算lint版本号，其他保持不变：

**lintVersion = gradlePluginVersion + 23.0.0**

例如，7 + 23 = 30，所以AGP版本_7.something_对应Lint版本_30.something_。作为另一个例子；截至撰写本文时，AGP的当前稳定版本是4.1.2，所以Lint API的相应版本是27.1.2。

为什么这个任意编号——为什么lint不能只使用相同的号码？这是历史原因；lint（和lint依赖的各种其他兄弟库）比Gradle插件更早发布；它已经到了版本22左右。当我们随后发布Gradle插件的初始版本与Android Studio 1.0时，我们想要为这个全新的工件从"1"开始重新编号。然而，一些其他库，如lint，不能只是从1重新开始，所以我们继续同步增加它们的版本。大多数用户看不到这个，但这是Lint API用户必须意识到的皱纹。

### :library和:app

`library`项目依赖于lint检查项目，并将lint检查作为其有效负载的一部分打包。`app`项目然后依赖于`library`，并有一些触发由此项目中的示例检查强制执行的lint检查的代码。这是为了演示如何发布和使用lint检查，这在发布Lint检查章节中详细描述。

### Lint检查项目布局

lint检查源项目非常简单：

```
checks/build.gradle
checks/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
checks/src/main/java/com/example/lint/checks/SampleIssueRegistry.kt
checks/src/main/java/com/example/lint/checks/SampleCodeDetector.kt
checks/src/test/java/com/example/lint/checks/SampleCodeDetectorTest.kt
```

首先是构建文件，我们已经在上面讨论过。

### 服务注册

然后是服务注册文件。注意这个文件在源集`src/main/resources/`中，这意味着Gradle将把它作为资源处理，并将其打包到输出jar中，在`META-INF/services`文件夹中。这使用了JDK中的服务提供者加载工具来注册lint可以查找的服务。键是lint的`IssueRegistry`类的完全限定名称。该文件的**内容**是一行，问题注册表的完全限定名称：

```
$ cat checks/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
com.example.lint.checks.SampleIssueRegistry
```

（服务加载器机制被IntelliJ理解，所以如果问题注册表被重命名等，它会正确更新服务文件内容。）

服务注册可以包含多个问题注册表，尽管通常没有好的理由这样做，因为单个问题注册表可以提供多个问题。

### IssueRegistry

接下来我们有从服务注册链接的`IssueRegistry`。Lint将实例化这个类并要求它提供问题列表。然后这些与lint的其他问题合并，当lint执行其分析时。

在最简单的形式中，我们只需要在该文件中有以下代码：

```kotlin
package com.example.lint.checks
import com.android.tools.lint.client.api.IssueRegistry
class SampleIssueRegistry : IssueRegistry() {
    override val issues = listOf(SampleCodeDetector.ISSUE)
}
```

然而，我们还提供了关于这些lint检查的一些额外元数据，如`Vendor`，它包含关于作者和（可选）联系地址或错误跟踪器信息的信息，当发现事件时显示给用户。

我们还提供了关于检查编译时使用的lint API版本以及此lint检查已测试的lint API最低版本的一些信息。（注意API版本与lint本身的版本不相同；想法和希望是API可能比提供新功能的lint更新演进得更慢。）

### Detector

`IssueRegistry`引用`SampleCodeDetector.ISSUE`，所以让我们看看`SampleCodeDetector`：

```kotlin
class SampleCodeDetector : Detector(), UastScanner {

    // ...

    companion object {
        /**
         * Issue describing the problem and pointing to the detector
         * implementation.
         */
        @JvmField
        val ISSUE: Issue = Issue.create(
            // ID: used in @SuppressLint warnings etc
            id = "SampleId",
            // Title -- shown in the IDE's preference dialog, as category headers in the
            // Analysis results window, etc
            briefDescription = "Lint Mentions",
            // Full explanation of the issue; you can use some markdown markup such as
            // `monospace`, *italic*, and **bold**.
            explanation = """
                    This check highlights string literals in code which mentions the word `lint`. \
                    Blah blah blah.

                    Another paragraph here.
                    """,
            category = Category.CORRECTNESS,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                SampleCodeDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

`Issue`注册非常不言自明，关于问题注册的详细信息在基础章节中涵盖。这里的过多注释是为了解释示例，通常在这样的问题注册代码中没有注释。

注意在第29行，`Issue`注册命名了负责分析此问题的`Detector`类：`SampleCodeDetector`。在上面我删除了该类的主体；现在这里是没有最后问题注册的主体：

```kotlin
package com.example.lint.checks

import com.android.tools.lint.client.api.UElementHandler
import com.android.tools.lint.detector.api.Category
import com.android.tools.lint.detector.api.Detector
import com.android.tools.lint.detector.api.Detector.UastScanner
import com.android.tools.lint.detector.api.Implementation
import com.android.tools.lint.detector.api.Issue
import com.android.tools.lint.detector.api.JavaContext
import com.android.tools.lint.detector.api.Scope
import com.android.tools.lint.detector.api.Severity
import org.jetbrains.uast.UElement
import org.jetbrains.uast.ULiteralExpression
import org.jetbrains.uast.evaluateString

class SampleCodeDetector : Detector(), UastScanner {
    override fun getApplicableUastTypes(): List<Class<out UElement?>> {
        return listOf(ULiteralExpression::class.java)
    }

    override fun createUastHandler(context: JavaContext): UElementHandler {
        return object : UElementHandler() {
            override fun visitLiteralExpression(node: ULiteralExpression) {
                val string = node.evaluateString() ?: return
                if (string.contains("lint") && string.matches(Regex(".*\\blint\\b.*"))) {
                    context.report(
                        ISSUE, node, context.getLocation(node),
                        "This code mentions `lint`: **Congratulations**"
                    )
                }
            }
        }
    }
}
```

这个lint检查非常简单；对于Kotlin和Java文件，它访问所有文字字符串，如果字符串包含单词"lint"，那么它发出警告。

这使用了AST分析的非常通用的机制；指定相关的节点类型（文字表达式，在第18行）并在第23行访问它们。Lint有大量的便利API来做更高级的事情，如"当有人扩展这个类时调用这个回调"，或"当有人调用名为`foo`的方法时"等。探索`SourceCodeScanner`和其他`Detector`接口，看看可能的情况。我们希望也会为此添加更多专门的文档。

### Detector测试

最后但同样重要的是，让我们不要忘记单元测试：

```kotlin
package com.example.lint.checks

import com.android.tools.lint.checks.infrastructure.TestFiles.java
import com.android.tools.lint.checks.infrastructure.TestLintTask.lint
import org.junit.Test

class SampleCodeDetectorTest {
    @Test
    fun testBasic() {
        lint().files(
            java(
                """
                package test.pkg;
                public class TestClass1 {
                    // In a comment, mentioning "lint" has no effect
                    private static String s1 = "Ignore non-word usages: linting";
                    private static String s2 = "Let's say it: lint";
                }
                """
            ).indented()
        )
        .issues(SampleCodeDetector.ISSUE)
        .run()
        .expect(
            """
            src/test/pkg/TestClass1.java:5: Warning: This code mentions lint: Congratulations [SampleId]
                private static String s2 = "Let's say it: lint";
                                           ∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼∼
            0 errors, 1 warnings
            """
        )
    }
}
```

如你所见，为lint编写单元测试非常简单，因为lint附带了专门的测试库；这是build.gradle中的

```groovy
testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
```

依赖引入的。

lint检查的单元测试在单元测试章节中深入涵盖，所以我们在这里将对上述测试的解释简短化。
