# Android Lint API 指南 - 中文翻译


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.html) 的中文翻译版本。

## 目录

1. [术语](#术语)
2. [编写Lint检查：基础知识](#编写lint检查基础知识)
3. [示例：GitHub示例Lint检查项目](#示例github示例lint检查项目)
4. [AST分析](#ast分析)
5. [发布Lint检查](#发布lint检查)
6. [Lint检查单元测试](#lint检查单元测试)
7. [测试模式](#测试模式)
8. [添加快速修复](#添加快速修复)
9. [部分分析](#部分分析)
10. [数据流分析器](#数据流分析器)
11. [注解](#注解)
12. [选项](#选项)
13. [错误消息约定](#错误消息约定)
14. [常见问题](#常见问题)

## 术语

在开始之前，你不需要完全理解所有内容，但这是一个便于回顾的参考。

按字母顺序排列：

### Configuration（配置）
配置为lint提供额外的信息或参数，基于每个项目甚至每个目录。例如，`lint.xml`文件可以更改问题的严重性，或列出要忽略的事件（例如通过正则表达式匹配），甚至为特定检测器读取的选项提供值。

### Context（上下文）
传递给检测器的对象，提供关于正在分析的文件（以及在哪个项目中）的数据，以及（对于特定类型的分析）附加信息；例如，XmlContext指向DOM文档，JavaContext包含AST等。

### Detector（检测器）
实现lint检查的类，注册Issues，分析代码并报告Incidents。

### Implementation（实现）
`Implementation`告诉lint如何实际分析给定的问题，比如要实例化哪个检测器类，以及检测器适用于哪些范围。

### Incident（事件）
问题在特定位置的特定发生。事件的例子是：
```
警告：在文件IoUtils.kt，第140行，字段下载文件夹
是"/sdcard/downloads"；不要硬编码路径到`/sdcard`。
```

### Issue（问题）
你的lint检查识别的问题类型或类别。问题有相关的严重性（错误、警告或信息）、优先级、类别、解释等。

问题的例子是"不要硬编码路径到/sdcard"。

### IssueRegistry（问题注册表）
`IssueRegistry`向lint提供问题列表。当你编写一个或多个lint检查时，你会在`IssueRegistry`中注册这些检查，并使用`META-INF`服务加载器机制指向它。

### LintClient
`LintClient`代表检测器运行的特定工具。例如，在IDE中运行时有一个LintClient（当报告事件时）会在编辑器中显示高亮，而当lint作为Gradle插件的一部分运行时，事件会被累积到HTML（和XML和文本）报告中，并在错误时中断构建。

### Location（位置）
"位置"指报告事件的地方。通常这指的是源文件中的文本范围，但位置也可以指向二进制文件，如`png`文件。位置也可以链接在一起，连同描述。因此，如果你要报告重复声明，你可以包含**两个**位置，在IDE中，两个位置（如果它们在同一文件中）都会被高亮。从另一个链接的位置称为"次要"位置，但链接可以任意长（lint的单元测试基础设施会确保没有循环）。

### Partial Analysis（部分分析）
lint中的"map reduce"架构，使得可以单独分析各个模块，然后根据这些模块之外的条件过滤和自定义部分结果。这在部分分析章节中有更详细的解释。

### Platform（平台）
`Platform`抽象允许lint问题指示它们适用的地方（如"Android"或"Server"等）。这意味着Android特定的检查不会在非Android代码上触发警告。

### Scanner（扫描器）
`Scanner`是检测器可以实现的特定接口，表示它支持一组特定的回调。例如，`XmlScanner`接口定义了访问XML元素和属性的方法，`ClassScanner`定义了ASM字节码处理方法等。

### Scope（范围）
`Scope`是一个枚举，列出了检测器可能想要分析的各种类型的文件。

例如，有XML文件的范围，有Java和Kotlin文件的范围，有.class文件的范围等。

通常lint关心哪**组**范围适用，所以大多数API采用`EnumSet<Scope>`，但我们通常称之为"范围"而不是"范围集"。

### Severity（严重性）
对于问题，事件应该是错误、警告还是都不是（只是FYI高亮）。还有一种特殊类型的错误严重性"致命"，稍后讨论。

### TextFormat（文本格式）
描述lint理解的各种文本格式的枚举。Lint检查通常只使用"raw"格式，这是类似markdown的格式（例如，你可以用星号包围单词使其斜体，两个星号使其粗体等）。

### Vendor（供应商）
`Vendor`是一个简单的数据类，提供关于lint检查来源的信息：谁写的，在哪里提交问题等。

## 编写Lint检查：基础知识

### 准备工作

（如果你已经知道很多基础知识，但因为遇到问题而查阅文档，请查看常见问题章节。）

#### "Lint？"

`lint`工具随C编译器一起提供，提供了超出编译器检查的C代码额外静态分析。

Android Lint以此工具命名，并加上Android前缀，明确表示这是一个用于分析Android代码的静态分析工具，由Android开源项目提供——并与许多其他名称中包含"lint"的工具区分。

然而，从那时起，Android Lint已经扩大了支持范围，不再仅用于Android代码。实际上，在Google内部，它用于分析所有Java和Kotlin代码。其中一个原因是它可以轻松分析Java和Kotlin代码，而无需实现两次检查。

我们计划重命名lint以反映这个新角色，所以我们正在寻找好的名称建议。

#### API稳定性

Lint的API不稳定，而且Lint的API表面的很大一部分不在我们的控制之下（如UAST和PSI）。因此，自定义lint检查可能需要定期更新以保持工作。

然而，"某些API比其他API更稳定"。特别是，检测器API（如下所述）比客户端API（不是为lint检查作者设计的，而是为集成lint运行的工具，如IDE和构建系统）更不可能改变。

然而，这并不意味着检测器API不会改变。API表面的很大一部分是lint外部的；它是来自JetBrains的Java和Kotlin的AST库（PSI和UAST）；它是字节码库（asm.ow2.io），它是XML DOM库（org.w3c.dom）等等。Lint有意与这些保持同步，所以这些中的任何API或行为变化都可能影响你的lint检查。

Lint自己的API也可能改变。当前API在过去10年中有机增长（lint的第一个版本于2011年发布），如果重新开始，有许多我们会清理和做不同的事情。更不用说重命名和清理不一致性。

然而，lint已经被广泛采用，所以在这一点上创建一个更好的API可能弊大于利，所以我们将最近的变化限制在必要的变化上。一个例子是7.0中的新部分分析架构，它允许更好的CI和增量分析性能。

#### Kotlin

我们建议你用Kotlin实现你的检查。部分原因是lint API使用了许多Kotlin特性：

- **命名和默认参数**：而不是使用构建器，一些构造方法，如`Issue.create()`有很多带默认参数的参数。如果你只指定你需要的并依赖其他所有默认值，API使用起来更清洁。
- **兼容性**：我们可能随时添加额外的参数。在所有地方添加@JvmOverloads是不实际的。
- **包级函数**：Lint的API包含许多包级实用函数（在API的以前版本中，这些都被扔到一个`LintUtils`类中）。
- **弃用**：Kotlin支持简单的API迁移。

### 概念

Lint将搜索你的源代码中的问题。有许多类型的问题，每一个称为`Issue`，它有相关的元数据，如唯一ID、类别、解释等。

它找到的每个实例称为"事件"。

搜索和报告事件的实际责任由检测器处理——`Detector`的子类。你的lint检查将扩展`Detector`，当它发现问题时，它将向lint"报告"事件。

`Detector`可以分析多个`Issue`。例如，内置的`StringFormatDetector`分析传递给`String.format()`调用的格式化字符串，在此过程中发现多个不相关的问题——无效的格式化字符串、应该使用复数API的格式化字符串、不匹配的类型等等。

`Detector`可以指示它关心哪些文件集。这些称为"范围"，工作方式是当你注册你的`Issue`时，你告诉该问题哪个`Detector`类负责分析它，以及检测器关心哪些范围。

例如，如果lint检查想要分析Kotlin文件，它可以包含`Scope.JAVA_FILE`范围，现在当lint处理Java或Kotlin文件时，该检测器将被包含。

名称`Scope.JAVA_FILE`可能听起来像也应该有`Scope.KOTLIN_FILE`。然而，这里的`JAVA_FILE`实际上指的是Java和Kotlin文件，因为对两者的分析和API是相同的（使用"UAST"，统一抽象语法树）。

当检测器实现各种回调时，它们可以分析代码，如果发现有问题的模式，可以"报告"事件。这意味着计算错误消息以及"位置"。事件的"位置"实际上是一个错误范围——文件，开始偏移和结束偏移。位置也可以链接在一起，所以例如对于"重复声明"错误，你可以并且应该包含两个位置。

许多检测器方法将传入`Context`，或`Context`的更具体子类，如`JavaContext`或`XmlContext`。这允许lint向检测器提供它们可能需要的信息，而不传入大量参数。它还允许lint随时添加额外数据而不破坏签名。

`Context`类也提供许多便利API。例如，对于`XmlContext`，有用于为XML标签、XML属性、XML属性的名称部分和XML属性的值部分创建位置的方法。对于`JavaContext`，也有创建位置的方法，如方法调用，包括是否包含接收器和/或参数列表。

当你报告`Incident`时，你也可以提供`LintFix`；这是IDE可以用来提供对警告采取行动的快速修复。在某些情况下，你可以提供完整和正确的修复（如删除未使用的元素）。在其他情况下，修复可能不太清楚；例如，`AccessibilityDetector`要求你为图像设置描述；快速修复将设置content属性，但将文本值留作TODO并选择字符串，以便用户可以只需输入来替换它。

报告事件时，确保错误消息不是通用的；尝试明确并包含当前场景的具体信息。例如，不要只是"重复声明"，使用"`$name`已经被声明"。这不仅仅是为了美观；它还使lint的基线机制工作得更好，因为它目前通过id + file + message匹配，而不是通常随时间漂移的行号。

### 客户端API与检测器API

Lint的API有两半：

- **客户端API**："从工具内集成（并运行）lint"。例如，IDE和构建系统都使用此API来嵌入和调用lint来分析项目或编辑器中的代码。
- **检测器API**："实现新的lint检查"。这是让检查器分析代码并报告它们发现的问题的API。

客户端API中代表在工具中运行的lint的类称为`LintClient`。这个类负责，除其他事项外：

- **报告检测器发现的事件**。例如，在IDE中，它会在源编辑器中放置错误标记，在构建系统中，它可能向控制台写入警告或生成报告或甚至使构建失败。
- **处理I/O**。检测器永远不应直接从磁盘读取文件。这允许lint检查在例如IDE中顺利工作。当lint实时运行时，lint检查请求源文件内容（或其他支持文件）时，IDE中的`LintClient`将实现`readFile`方法，首先查看打开的源编辑器，如果请求的文件正在编辑，它将返回当前（通常未保存！）内容。
- **处理网络流量**。Lint检查永远不应自己打开URLConnection。相反，它们应该通过lint API请求URL的数据。

### 创建Issue

`Issue`是最终类，所以不像`Detector`，你不子类化它；你通过`Issue.create`实例化它。

按约定，问题在相应检测器的伴生对象内注册，但这不是必需的。

这是一个例子：

```kotlin
class SdCardDetector : Detector(), SourceCodeScanner {
    companion object Issues {
        @JvmField
        val ISSUE = Issue.create(
            id = "SdCardPath",
            briefDescription = "Hardcoded reference to `/sdcard`",
            explanation = """
                Your code should not reference the `/sdcard` path directly; \
                instead use `Environment.getExternalStorageDirectory().getPath()`.

                Similarly, do not reference the `/data/data/` path directly; it \
                can vary in multi-user scenarios. Instead, use \
                `Context.getFilesDir().getPath()`.
                """,
            moreInfo = "https://developer.android.com/training/data-storage#filesExternal",
            category = Category.CORRECTNESS,
            severity = Severity.WARNING,
            androidSpecific = true,
            implementation = Implementation(
                SdCardDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
    // ...
}
```

有几个值得注意的事情。

**`id`**是此问题的短、唯一标识符。按约定，它是单词的组合，大写驼峰命名（虽然你也可以添加自己的包前缀，如Java包）。注意id是"用户可见的"；当lint在构建系统中运行时，它包含在文本输出中。

ID对用户可见的原因是ID是他们配置和/或抑制问题的方式。例如，要在当前方法中抑制警告，使用：

```kotlin
@Suppress("SdCardPath")
```

（或在Java中，@SuppressWarnings）。

接下来，我们有**`briefDescription`**。你可以将此视为"类别报告标题"；这是此类型所有事件的静态描述，因此不能包含任何具体信息。此字符串例如用作HTML报告中此类型所有事件的标题，在IDE中，如果你打开Inspections UI，各种问题使用简短描述列出。

**`explanation`**是对问题是什么的多行、理想情况下多段解释。在某些情况下，问题是不言自明的，如"未使用声明"的情况，但在许多情况下，问题更微妙，可能需要额外解释，特别是开发人员应该**做什么**来解决问题。解释包含在HTML报告和IDE检查结果窗口中。

**`category`**不是超级重要的；主要用途是类别名称可以在问题配置时被视为ID；例如，用户可以关闭所有国际化问题，或仅针对安全相关问题运行lint。类别也用于在HTML报告中定位相关问题。如果没有内置类别合适，你也可以创建自己的。

**`severity`**属性非常重要。问题可以是警告或错误。这些在IDE中（错误是红色下划线，警告是黄色高亮）和构建系统中（错误可以选择性地破坏构建，警告不会）被不同对待。

你也可以指定**`moreInfo`** URL，它将作为"更多信息"链接包含在问题解释中，以打开阅读关于此问题或底层问题的更多详细信息。

### TextFormat

lint中的所有错误消息和问题元数据字符串都使用简单的类似Markdown的语法解释：

| 原始文本格式 | 渲染为 |
|------------|--------|
| This is a \`code symbol\` | This is a code symbol |
| This is \*italics\* | This is *italics* |
| This is \*\*bold\*\* | This is **bold** |

这在Lint生成的HTML报告或IDE中显示错误消息和问题解释时很有用，例如错误消息工具提示将使用格式化。

### Issue实现

最后的问题注册属性是**`implementation`**。这是我们将元数据粘合到我们的特定分析器实现的地方，该实现可以找到此问题的实例。

通常，`Implementation`提供两个东西：

- 应该被实例化的我们的`Detector`的`.class`。在上面的代码示例中是`SdCardDetector`。
- 此问题的检测器适用的`Scope`。在上面的例子中是`Scope.JAVA_FILE`，这意味着它将适用于Java和Kotlin文件。

### 范围

`Implementation`实际上采用**一组**范围；我们仍然称之为"范围"。一些lint检查想要分析多种类型的文件。例如，`StringFormatDetector`将分析声明各种语言环境格式化字符串的资源文件，以及包含引用格式化字符串的`String.format`调用的Java和Kotlin文件。

`Scope`类中有许多预定义的范围集。`Scope.JAVA_FILE_SCOPE`是最常见的，它是包含确切`Scope.JAVA_FILE`的单例集，但你总是可以创建自己的。

当lint问题需要多个范围时，这意味着lint将**只有**在运行工具中**所有**范围都可用时运行此检测器。当lint运行完整批处理运行（如Gradle lint目标或IDE中的完整"检查代码"）时，所有范围都可用。

然而，当lint在编辑器中实时运行时，它只能访问当前文件；它不会为每几次击键重新分析项目中的_所有_文件。所以在这种情况下，lint驱动程序中的范围只包括当前源文件的类型，只有指定子集范围的lint检查才会运行。

这是新lint检查作者的常见错误：lint检查作为单元测试工作得很好，但他们在IDE中看不到工作，因为问题实现请求多个范围，**所有**范围都必须可用。

通常，lint检查查看多个源文件类型以在所有情况下正确工作，但在给定单个源文件的情况下仍然可以识别_某些_问题。在这种情况下，`Implementation`构造函数（采用范围集的vararg）可以被传递额外的范围集，称为"分析范围"。如果当前lint客户端的范围匹配或是任何分析范围的子集，那么检查将运行。

### 注册Issue

一旦你创建了你的问题，你需要从`IssueRegistry`提供它。

这是一个`IssueRegistry`的例子：

```kotlin
package com.example.lint.checks

import com.android.tools.lint.client.api.IssueRegistry
import com.android.tools.lint.client.api.Vendor
import com.android.tools.lint.detector.api.CURRENT_API

class SampleIssueRegistry : IssueRegistry() {
    override val issues = listOf(SdCardDetector.ISSUE)

    override val api: Int
        get() = CURRENT_API

    override val minApi: Int
        get() = 8

    override val vendor: Vendor = Vendor(
        vendorName = "Android Open Source Project",
        feedbackUrl = "https://com.example.lint.blah.blah",
        contact = "author@com.example.lint"
    )
}
```

我们正在返回我们的问题。它是一个列表，所以`IssueRegistry`可以提供多个问题。

**`api`**属性应该完全像上面出现的方式在你自己的问题注册表中编写；这将记录此问题注册表编译所针对的lint API版本。

**`minApi`**属性记录此检查已测试的最旧lint API级别。

**`vendor`**属性是7.0的新功能，为lint作者提供了一种指示lint检查来自何处的方法。

使lint检查可用的最后一步是通过服务加载器机制使`IssueRegistry`已知。

创建一个确切命名的文件

```
src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
```

内容如下（但你要替换为你自己的问题注册表的完全限定类名）：

```
com.example.lint.checks.SampleIssueRegistry
```

### 实现Detector：扫描器

我们终于来到了编写lint检查的主要任务：实现**`Detector`**。

这是一个微不足道的：

```kotlin
class MyDetector : Detector() {
   override fun run(context: Context) {
       context.report(ISSUE, Location.create(context.file),
           "I complain a lot")
   }
}
```

这将在每个文件中抱怨。显然，没有真正的lint检测器这样做；我们想要做一些分析并**有条件地**报告事件。

为了使分析更简单，Lint专门支持分析各种文件类型。工作方式是你注册兴趣，然后调用各种回调。

例如：

- 实现**`XmlScanner`**时，在XML元素中你可以被回调
  - 当声明任何给定标签集时（`visitElement`）
  - 当声明任何命名属性集时（`visitAttribute`）
  - 你可以通过`visitDocument`执行自己的文档遍历
- 实现**`SourceCodeScanner`**时，在Kotlin和Java文件中你可以被回调
  - 当调用给定名称的方法时（`getApplicableMethodNames`和`visitMethodCall`）
  - 当实例化给定类型的类时（`getApplicableConstructorTypes`和`visitConstructor`）
  - 当声明扩展（可能间接）给定类或接口的新类时（`applicableSuperClasses`和`visitClass`）
  - 当引用或组合注解元素时（`applicableAnnotations`和`visitAnnotationUsage`）
  - 当出现给定类型的任何AST节点时（`getApplicableUastTypes`和`createUastHandler`）

### Detector生命周期

Detector注册是按检测器类而不是按检测器实例完成的。Lint将代表你实例化检测器。它将为每次分析实例化检测器一次，所以你可以在字段中存储检测器状态并累积信息以在最后进行分析。

在分析每个单独文件之前和之后都有一些回调（`beforeCheckFile`和`afterCheckFile`），以及在分析所有模块之前和之后（`beforeCheckRootProject`和`afterCheckRootProject`）。

这例如是"未使用资源"检查的工作方式：我们在处理每个文件时存储我们在项目中找到的所有资源声明和资源引用，然后在`afterCheckRootProject`方法中分析资源图并计算在引用图中不可达的任何资源声明，然后我们将每个报告为未使用。

### 扫描器顺序

一些lint检查涉及多个扫描器。这在Android中很常见，我们想要交叉检查资源文件中的数据与代码使用之间的一致性。例如，`String.format`检查确保传递给`String.format`的参数与所有翻译XML文件中指定的格式化字符串匹配。

Lint定义了处理扫描器的确切顺序，在扫描器内，数据。这使得编写一些检测器更容易，因为你知道你会在另一个之前遇到一种类型的数据；你不必处理相反的顺序。

这是lint的定义顺序：

1. Android清单
2. Android资源XML文件（按文件夹类型字母顺序，所以例如布局在值文件如翻译之前处理）
3. Kotlin和Java文件
4. 字节码（本地`.class`文件和库`.jar`文件）
5. TOML文件
6. Gradle文件
7. 其他文件
8. ProGuard文件
9. 属性文件

同样，lint总是在依赖它们的模块之前处理库。

### 实现Detector：服务

除了扫描器，lint提供许多服务来使实现更简单。这些包括

- **`ConstantEvaluator`**：执行AST表达式的评估
- **`TypeEvaluator`**：尝试提供表达式的具体类型
- **`JavaEvaluator`**：尽管名称不幸较老，此服务适用于Kotlin和Java，可以例如提供关于继承层次结构、从完全限定名称查找类等的信息
- **`DataFlowAnalyzer`**：方法内的数据流分析
- 对于Android分析，有几个其他重要服务，如`ResourceRepository`和`ResourceEvaluator`
- 最后，有许多实用方法；例如有用于找到可能拼写错误的`editDistance`方法

### 扫描器示例

让我们使用上述扫描器之一`XmlScanner`创建一个`Detector`，它将查看项目中的所有XML文件，如果遇到`<bitmap>`标签，它将报告应该使用`<vector>`：

```kotlin
import com.android.tools.lint.detector.api.Detector
import com.android.tools.lint.detector.api.Detector.XmlScanner
import com.android.tools.lint.detector.api.Location
import com.android.tools.lint.detector.api.XmlContext
import org.w3c.dom.Element

class MyDetector : Detector(), XmlScanner {
    override fun getApplicableElements() = listOf("bitmap")

    override fun visitElement(context: XmlContext, element: Element) {
        val incident = Incident(context, ISSUE)
            .message( "Use `<vector>` instead of `<bitmap>`")
            .at(element)
        context.report(incident)
    }
}
```

### 分析Kotlin和Java代码

#### UAST

要分析Kotlin和Java代码，lint提供代码的抽象语法树或"AST"。

这个AST称为"UAST"，代表"通用抽象语法树"，它以相同方式表示多种语言，隐藏语言特定细节，如语句末尾是否有分号或注解类的声明方式是`@interface`还是`annotation class`等。

这使得可以编写跨UAST支持的所有语言工作的单个分析器。这非常有用；大多数lint检查做的是API或数据流特定的事情，而不是语言特定的事情。

在UAST中，每个元素称为**`UElement`**，有许多子类——`UFile`用于编译单元，`UClass`用于类，`UMethod`用于方法，`UExpression`用于表达式，`UIfExpression`用于`if`表达式等。

名称"UAST"有点误导；它不是所有可能语法树的某种超集；相反，将此视为所有代码的"Java视图"。

#### UAST示例

这是一个例子（来自内置的Android `AlarmDetector`），显示了上述所有实践；这是一个lint检查，确保如果有人调用`AlarmManager.setRepeating`，第二个参数至少是5,000，第三个参数至少是60,000。

```kotlin
override fun getApplicableMethodNames(): List<String> = listOf("setRepeating")

override fun visitMethodCall(
    context: JavaContext,
    node: UCallExpression,
    method: PsiMethod
) {
    val evaluator = context.evaluator
    if (evaluator.isMemberInClass(method, "android.app.AlarmManager") &&
        evaluator.getParameterCount(method) == 4
    ) {
        ensureAtLeast(context, node, 1, 5000L)
        ensureAtLeast(context, node, 2, 60000L)
    }
}
```

### 查找UAST

要编写检测器的分析，你需要知道你感兴趣的代码的AST是什么样的。不是试图通过在调试器下检查元素来弄清楚，一个简单的方法是"漂亮打印"它，使用`UElement`扩展方法**`asRecursiveLogString`**。

### 解析

当你有方法调用或字段引用时，你可能想要查看被调用的方法或字段。这称为"解析"，UAST直接支持它；例如在`UCallExpression`上调用`.resolve()`，它返回`PsiMethod`。

解析只有在lint有正确类路径以便引用的方法、字段或类实际存在时才起作用。如果不存在，resolve将返回null，各种lint回调将不被调用。

### 隐式调用

Kotlin支持许多内置运算符的运算符重载。例如，如果你有以下代码：

```kotlin
fun test(n1: BigDecimal, n2: BigDecimal) {
    // 这里，这实际上是对BigDecimal#compareTo的中缀调用
    if (n1 < n2) {
        // ...
    }
}
```

这里的`<`实际上是一个函数调用。

然而，注意在抽象语法树中，这**不是**表示为`UCallExpression`；这里我们将有一个`UBinaryExpression`，左操作数`n1`，右操作数`n2`和运算符`UastBinaryOperator.LESS`。

Lint有一些特殊支持来帮助处理这些情况。

首先，对调用回调的内置支持（你通过从`getApplicableMethodNames`返回名称注册对调用名称的兴趣，然后在`visitMethodCall`回调中响应）已经自动处理这个。

### PSI

PSI是"程序结构接口"的缩写，是IntelliJ在IDE中用于所有语言建模的AST抽象。

注意每种语言都有**不同的**PSI表示。Java和Kotlin涉及完全不同的PSI类。这意味着使用PSI编写lint检查将涉及编写很多逻辑两次；一次用于Java，一次用于Kotlin。

这就是UAST的用途：有从Java PSI到UAST的"桥接"，有从Kotlin PSI到UAST的桥接，你的lint检查只分析UAST。

然而，有几种情况我们必须使用PSI。

第一个，也是最常见的，在前面关于解析的部分列出。UAST不完全替代PSI；实际上，PSI在UAST API表面的一部分泄漏。例如，`UMethod.resolve()`返回`PsiMethod`。更重要的是，`UMethod` **扩展** `PsiMethod`。

### 测试

为lint检查编写单元测试很重要，这在专门的单元测试章节中详细介绍。

## 示例：GitHub示例Lint检查项目

<https://github.com/googlesamples/android-custom-lint-rules> GitHub项目提供了一个显示工作骨架的示例lint检查。

本章介绍该示例项目并解释什么和为什么。

### 项目布局

这是示例项目的项目布局：

```
implementation ← lintPublish
:app ← :library ← :checks
```

我们有一个应用程序模块`app`，它依赖（通过`implementation`依赖）一个`library`，库本身对`checks`项目有`lintPublish`依赖。

### :checks

`checks`项目是实际实现lint检查的地方。这个项目是一个纯Kotlin或纯Java Gradle项目：

```gradle
apply plugin: 'java-library'
apply plugin: 'kotlin'
```

Gradle文件还声明了我们的检测器需要的lint API的依赖：

```gradle
dependencies {
    compileOnly "com.android.tools.lint:lint-api:$lintVersion"
    compileOnly "com.android.tools.lint:lint-checks:$lintVersion"
    testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
}
```

### lintVersion？

上面定义的`lintVersion`变量是什么？

当你构建lint检查时，你正在编译针对在maven.google.com上分发的Lint API（在Gradle文件中通过`google()`引用）。这些遵循Gradle插件版本号。

因此，你首先选择你想要编译的lint API。你应该尽可能使用最新可用的。

一旦你知道Gradle插件版本号，比如4.2.0-beta06，你可以通过简单地将**23**添加到gradle插件的主要版本来计算lint版本号，其他保持不变：

**lintVersion = gradlePluginVersion + 23.0.0**

例如，7 + 23 = 30，所以AGP版本_7.something_对应Lint版本_30.something_。

### Lint检查项目布局

lint检查源项目非常简单：

```
checks/build.gradle
checks/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
checks/src/main/java/com/example/lint/checks/SampleIssueRegistry.kt
checks/src/main/java/com/example/lint/checks/SampleCodeDetector.kt
checks/src/test/java/com/example/lint/checks/SampleCodeDetectorTest.kt
```

### 服务注册

然后是服务注册文件。注意这个文件在源集`src/main/resources/`中，这意味着Gradle将把它作为资源处理并将其打包到输出jar中，在`META-INF/services`文件夹中。

### IssueRegistry

接下来我们有从服务注册链接的`IssueRegistry`。Lint将实例化这个类并要求它提供问题列表。

### Detector

`IssueRegistry`引用`SampleCodeDetector.ISSUE`，所以让我们看看`SampleCodeDetector`：

这个lint检查非常简单；对于Kotlin和Java文件，它访问所有文字字符串，如果字符串包含单词"lint"，那么它发出警告。

### Detector测试

最后但并非最不重要的，让我们不要忘记单元测试：

正如你所看到的，为lint检查编写单元测试非常简单，因为lint附带专用的测试库。

## AST分析

要分析Kotlin和Java文件，lint提供许多便利回调，使完成常见任务变得简单：

- 检查对特定方法名称的调用
- 实例化特定类
- 扩展特定超类或接口
- 使用特定注解，或调用注解方法

但在某些情况下，你需要深入并自己分析"AST"。

### AST分析

AST是"抽象语法树"的缩写——源代码的树表示。

### UAST

我们可以构造一个表示相同概念的新AST：

这是一个统一的AST，在称为"UAST"的东西中，UAST的缩写是统一抽象语法树。UAST是我们在Lint中用于代码的主要AST表示。

### UAST：Java视图

注意这里名称中的"统一"有点误导。从名称你可能假设这是跨语言的某种AST超集——可以表示所有语言所需一切的AST。但情况并非如此！相反，更好的思考方式是作为AST的**Java视图**。

### 表达式

你可能得到UAST树非常浅且只表示高级声明（如文件、类、方法和属性）的印象。

情况并非如此。虽然它**确实**跳过低级、语言特定细节，如空白节点和单个关键字节点，所有各种表达式类型都被表示并可以推理。

### UElement

UAST中的每个节点都是`UElement`的子类。有一个父指针，这对在AST中导航很方便。

编写lint检查你需要的真正技能是理解AST，然后对其进行模式匹配。一个简单的技巧是在单元测试中创建你想要的Kotlin或Java代码，然后在你的检测器中，递归打印UAST作为树。

或在调试器中，任何时候你有`UElement`，你可以在其上调用`UElement.asRecursiveLogString`，评估并看看你找到什么。

### 访问

你通常不应该自己访问源文件。Lint有一个特殊的`UElementHandler`，用于确保我们不重复访问源文件数千次，每个检测器一次。

但当你做本地分析时，你有时需要访问子树。

### UElement到PSI映射

UAST建立在PSI之上，每个`UElement`都有一个`sourcePsi`属性（可能为null）。这让你从通用UAST节点映射到特定PSI元素。

### PSI到UElement

在某些情况下，我们也可以使用`toUElement`扩展函数从PSI元素映射回UAST。

### UAST与PSI

UAST是在为Kotlin和Java编写lint检查时使用的首选AST。它让你推理跨语言相同的事情。声明。函数调用。超类。赋值。If表达式。Return语句。等等。

但虽然你通常想要使用UAST进行分析（lint的API通常面向UAST），**有**适合深入PSI的情况。

特别是，当你做高度语言特定的事情，语言细节在UAST中没有暴露时，你应该使用PSI。

### Kotlin分析API

使用Kotlin PSI是正确分析Kotlin代码的技术水平，直到最近。但当你查看PSI时，你会发现一些事情真的很难完成——特别是解析引用和处理Kotlin类型。

对于大多数lint检查，如果可以的话你应该只使用UAST。但当你需要了解真正**详细的**Kotlin信息，特别是关于类型、智能转换和null推断等时，Kotlin分析API是你最好的朋友（也是唯一选择...）

## 发布Lint检查

Lint将查找带有问题注册表服务注册表键的jar文件。

你可以通过使用环境变量ANDROID_LINT_JARS手动指向你的自定义lint检查jar文件：

```bash
$ export ANDROID_LINT_JARS=/path/to/first.jar:/path/to/second.jar
```

然而，这只用于开发和作为不直接支持lint或嵌入式lint库的构建系统的解决方法。

### Android

#### AAR支持

Android库作为`.aar`文件而不是`.jar`文件发布。这意味着它们可以携带不仅仅是代码负载。在底层，`.aar`文件只是包含许多其他嵌套文件的zip文件，包括api和实现jar、资源、proguard/r8规则，是的，lint jar。

这种方法的优势是当lint注意到你依赖一个库，该库包含自定义lint检查时，lint将拉入这些检查并应用它们。这为库作者提供了一种提供自己的额外检查强制使用的方式。

#### lintPublish配置

Android Gradle库插件提供一些特殊配置，`lintChecks`和`lintPublish`。

`lintPublish`配置让你引用另一个项目，它将获取该项目的输出jar并将其打包为AAR文件内的`lint.jar`。

## Lint检查单元测试

### 创建单元测试

为lint检查编写单元测试的测试基础设施内置在lint中。

测试lint检查的基本方法是扩展`LintDetectorTest`：

```kotlin
class MyDetectorTest : LintDetectorTest() {
    fun testBasic() {
        lint().files(
            java("""
                package test.pkg;
                public class MyTest {
                    public void test() {
                        // 这里是触发检查的代码
                    }
                }
                """).indented()
        ).issues(MyDetector.ISSUE).run().expect("""
            src/test/pkg/MyTest.java:4: Warning: ... [MyIssue]
                // 预期的错误输出
            0 errors, 1 warnings
            """)
    }
}
```

### 计算预期输出

不是试图手动弄清楚预期输出应该是什么，一个简单的方法是运行测试，让它失败，然后从失败消息复制实际输出到预期输出中。

### 测试文件

在上面的例子中，我们使用`java(...)`创建了一个测试Java文件。有许多其他类似方法：

- `kotlin(...)`：创建Kotlin文件
- `xml(...)`：创建XML文件
- `manifest(...)`：创建AndroidManifest.xml文件
- `gradle(...)`：创建build.gradle文件

### 库依赖和存根

因为`LintDetectorTest` API无法访问库类和方法，你必须为任何必要的类实现存根并将这些作为测试用例中的额外文件包含。

## 测试模式

### 如何调试

### 处理故意失败

### 源修改测试模式

## 添加快速修复

### 介绍

### LintFix构建器类

### 创建LintFix

### 可用修复

### 组合修复

### 重构Java和Kotlin代码

## 部分分析

### 关于

### 问题

### 概述

### 我的检测器需要工作吗？

### 事件

### 约束

## 数据流分析器

### 使用

### 自引用调用

### Kotlin作用域函数

### 限制

### 转义值

### 非本地分析

### 示例

## 注解

### 基础

### 注解使用类型和isApplicableAnnotationUsage

### 方法重写

### 方法返回

### 处理使用类型

## 选项

### 使用

### 创建选项

### 读取选项

### 特定配置

### 文件

### 约束

### 测试选项

## 错误消息约定

### 长度

错误消息应该简洁。作为一般规则，尝试保持在80个字符以下，绝对不超过120个字符。

### 格式化

错误消息经常引用类名和方法名，这些应该用反引号包围。

### 标点

错误消息不应以句号结尾。

### 包含细节

错误消息应该具体而不是通用。不要只说"重复声明"，说"`$name`已经被声明"。

### 引用Android版本号

当引用Android API级别时，包含版本号和代号：

```
This feature requires API level 21 (Android 5.0)
```

### 保持消息稳定

不要频繁更改错误消息文本，因为这会破坏基线文件。

### 复数

当引用可能是复数的事物时，正确处理复数形式。

### 示例

好的错误消息示例：

```
"Use `ViewCompat.setBackground()` instead of `setBackgroundDrawable()`"
"Missing `contentDescription` attribute on image"
"Hardcoded string \"Hello\", should use `@string` resource"
```

不好的错误消息示例：

```
"Invalid."
"Wrong"
"This is not correct and you should fix it by doing something else instead"
```

## 常见问题

### 我的检测器回调没有被调用

确保你的问题注册正确指定了范围。

### 我的lint检查在单元测试中工作但在IDE中不工作

这通常是因为范围问题。IDE只有当前文件的范围。

### visitAnnotationUsage没有为注解调用

确保你正确实现了`isApplicableAnnotationUsage`。

### 如何检查UAST或PSI元素是Java还是Kotlin？

你可以检查`sourcePsi`的类型。

### 如果我有UElement但需要PsiElement怎么办？

使用`sourcePsi`属性。

### 如何为PsiMethod获取UMethod？

使用`toUElement()`扩展函数。

### 如何获取JavaEvaluator？

从context：`context.evaluator`。

### 如何检查元素是否为内部？

对于Kotlin，检查可见性修饰符。

### 元素是否为inline、sealed、operator、infix、suspend、data？

你需要检查Kotlin PSI修饰符。

### 如果我有完全限定名称，如何查找类？

使用`JavaEvaluator.findClass()`。

### 如果我有PsiType，如何查找类？

使用`JavaEvaluator.getTypeClass()`。

### 如何为元素查找层次结构注解？

使用`JavaEvaluator.getAllAnnotations()`。

### 如何查找类是否是另一个类的子类？

使用`JavaEvaluator.extendsClass()`。

### 如何知道调用参数对应哪个参数？

使用`JavaEvaluator.computeArgumentMapping()`。

### 我的lint检查如何针对两个不同版本的lint？

使用反射或版本检查。

### 我可以让我的lint检查"不可抑制"吗？

不，所有lint检查都应该是可抑制的。

### 为什么重载运算符没有被处理？

确保你正在寻找隐式调用。

### 如何查看当前lint源代码？

访问Android源代码存储库。

### 在哪里找到lint检查的例子？

查看内置检查或GitHub示例。

### 如何分析Kotlin的详细信息？

使用Kotlin分析API。

## 附录：最近的变化

Lint API在不断演进中。以下是一些重要的最近变化：

### 7.0版本变化

- 引入了新的部分分析架构
- 添加了`Vendor`属性支持
- 改进了`Incident`报告API
- 增强了快速修复功能

### API兼容性

- Lint API不是稳定的，可能在未来版本中发生变化
- 建议使用最新版本的API进行开发
- 旧版本的检查器可能需要更新以保持兼容性

### 迁移指南

从旧版本迁移到新版本时，请注意以下几点：

- 检查已弃用的API并替换为新的API
- 更新`IssueRegistry`中的API版本号
- 测试检查器在新版本中的工作情况

## 附录：环境变量和系统属性

### 环境变量

#### 检测器配置变量

- `ANDROID_LINT_JARS`：指定自定义lint检查jar文件的路径
- `LINT_OVERRIDE_CONFIGURATION`：覆盖默认配置设置

#### Lint配置变量

- `LINT_XML_ROOT`：指定lint.xml配置文件的根目录
- `LINT_BASELINE_FILE`：指定基线文件的路径

#### Lint开发变量

- `LINT_TEST_KOTLINC`：指定kotlinc编译器的路径，用于测试
- `LINT_API_DEBUG`：启用API调试模式
- `LINT_PERFORMANCE`：启用性能分析

### 系统属性

- `lint.use.fir.uast`：启用K2 UAST支持
- `lint.autofix`：启用自动修复模式
- `lint.debug`：启用调试输出
- `lint.html.prefs`：HTML报告的偏好设置

### 配置示例

```bash
# 设置自定义lint检查
export ANDROID_LINT_JARS="/path/to/custom/lint.jar"

# 启用K2支持
java -Dlint.use.fir.uast=true -jar lint.jar

# 启用调试模式
java -Dlint.debug=true -jar lint.jar
```

这些环境变量和系统属性可以帮助你在开发和测试自定义lint检查时进行更精细的控制和调试。

---

## 总结

本指南为Android Lint自定义规则开发提供了全面的中文参考文档，涵盖了从基础概念到高级主题的所有内容。通过学习本指南，你应该能够：

1. **理解Lint的核心概念**：掌握Issue、Detector、Scanner等基本概念
2. **编写自定义检查**：创建适合项目需求的自定义lint规则
3. **进行AST分析**：使用UAST和PSI分析Java和Kotlin代码
4. **实现高级功能**：添加快速修复、处理注解、使用数据流分析器等
5. **测试和调试**：编写单元测试并调试自定义检查
6. **发布和应用**：将自定义检查集成到项目中

Lint是一个强大的静态分析工具，通过编写自定义规则，你可以为项目添加特定的代码质量检查，提高代码的可维护性和一致性。

**注意**：由于Lint API不是稳定的，在使用时请关注API的变化，并及时更新你的检查器以保持兼容性。

**参考资源**：
- [Android Lint官方文档](https://developer.android.com/studio/write/lint)
- [Google Sample项目](https://github.com/googlesamples/android-custom-lint-rules)
- [Lint API JavaDoc](https://www.javadoc.io/doc/com.android.tools.lint/lint-api)


