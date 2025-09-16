# Android Lint API 指南 - 第2章：编写Lint检查基础知识


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第2章节"Writing a Lint Check: Basics"的中文翻译版本。

## 编写Lint检查：基础知识

### 前言

（如果你已经了解很多基础知识但因为遇到问题而查阅文档，请查看常见问题章节。）

#### "Lint？"

`lint`工具与C编译器一起提供，并提供C代码的额外静态分析，超出了编译器检查的范围。

Android Lint以此工具为名，并加上Android前缀以明确表示这是一个用于分析Android代码的静态分析工具，由Android开源项目提供——并与许多其他名称中包含"lint"的工具区分开来。

然而，从那时起，Android Lint已经扩大了其支持范围，不再仅仅用于Android代码。事实上，在Google内部，它被用来分析所有Java和Kotlin代码。其中一个原因是它可以轻松分析Java和Kotlin代码，而无需实现两次检查。其他功能在功能章节中描述。

我们计划重命名lint以反映这个新角色，所以我们正在寻找好的名称建议。

#### API稳定性

Lint的API不稳定，而且Lint的API表面的很大一部分不在我们的控制之下（如UAST和PSI）。因此，自定义lint检查可能需要定期更新以保持工作。

然而，"某些API比其他API更稳定"。特别是，检测器API（如下所述）比客户端API（不适用于lint检查作者，而是用于集成lint在其中运行的工具，如IDE和构建系统）更不可能改变。

然而，这并不意味着检测器API不会改变。API表面的很大一部分是lint外部的；它是来自JetBrains的Java和Kotlin的AST库（PSI和UAST）；它是字节码库（asm.ow2.io），它是XML DOM库（org.w3c.dom）等等。Lint有意与这些保持同步，所以这些中的任何API或行为变化都可能影响你的lint检查。

Lint自己的API也可能改变。当前的API在过去10年中有机地增长（lint的第一个版本于2011年发布），如果重新开始，我们会清理和做不同的事情。更不用说重命名和清理不一致性了。

然而，lint已经被广泛采用，所以在这一点上创建一个更好的API可能弊大于利，所以我们将最近的变化限制在必要的变化上。这方面的一个例子是7.0中的新部分分析架构，它允许更好的CI和增量分析性能。

#### Kotlin

我们建议你在Kotlin中实现你的检查。部分原因是lint API使用了许多Kotlin功能：

* **命名和默认参数**：而不是使用构建器，一些构造方法，如`Issue.create()`有很多带有默认参数的参数。如果你只指定你需要的内容并依赖其他所有内容的默认值，API使用起来更清洁。
* **兼容性**：我们可能会随时间添加其他参数。在所有内容上添加@JvmOverloads是不实际的。
* **包级函数**：Lint的API包括许多包级实用函数（在以前版本的API中，这些都被放在`LintUtils`类中）。
* **弃用**：Kotlin支持简单的API迁移。例如，在下面的示例中，第1到7行的新`@Deprecated`注解将在即将发布的版本中添加，以简化向新API的迁移。IntelliJ可以自动快速修复这些弃用替换。

```kotlin
@Deprecated(
    "Use the new report(Incident) method instead, which is more future proof",
    ReplaceWith(
        "report(Incident(issue, message, location, null, quickfixData))",
        "com.android.tools.lint.detector.api.Incident"
    )
)
@JvmOverloads
open fun report(
    issue: Issue,
    location: Location,
    message: String,
    quickfixData: LintFix? = null
) {
    // ...
}
```

从7.0开始，lint中的Kotlin代码比剩余的Java代码更多：

| 语言 | 文件 | 空白 | 注释 | 代码 |
|------|------|------|------|------|
| Kotlin | 420 | 14243 | 23239 | 130250 |
| Java | 289 | 8683 | 15205 | 101549 |

`$ cloc lint/`

这是对所有lint的统计，包括许多多年未触及的旧lint检测器。在Lint API库`lint/libs/lint-api`中，代码是78% Kotlin和22% Java。

### 概念

Lint将搜索你的源代码中的问题。有许多类型的问题，每一个都称为`Issue`，它具有关联的元数据，如唯一ID、类别、说明等。

它找到的每个实例称为"事件"。

搜索和报告事件的实际责任由检测器处理——`Detector`的子类。你的lint检查将扩展`Detector`，当它发现问题时，它将向lint"报告"事件。

`Detector`可以分析多个`Issue`。例如，内置的`StringFormatDetector`分析传递给`String.format()`调用的格式字符串，在这个过程中发现多个不相关的问题——无效的格式字符串、应该使用复数API的格式字符串、不匹配的类型等等。检测器可以简单地有一个名为"StringFormatProblems"的单一问题并将所有内容报告为StringFormatProblem，但这不是一个好主意。每个这些单独类型的字符串格式问题应该有自己的解释、自己的类别、自己的严重性，最重要的是应该由用户单独配置，以便他们可以分别禁用或提升这些问题中的一个。

`Detector`可以指示它关心哪些文件集。这些称为"作用域"，工作方式是当你注册你的`Issue`时，你告诉该问题哪个`Detector`类负责分析它，以及检测器关心哪些作用域。

如果例如一个lint检查想要分析Kotlin文件，它可以包含`Scope.JAVA_FILE`作用域，现在当lint处理Java或Kotlin文件时，该检测器将被包含。

名称`Scope.JAVA_FILE`可能听起来应该也有一个`Scope.KOTLIN_FILE`。然而，这里的`JAVA_FILE`实际上指的是Java和Kotlin文件，因为分析和API对两者都是相同的（使用"UAST"，统一的抽象语法树）。然而，在这一点上我们不想重命名它，因为它会破坏许多现有的检查。我们可能在将来引入别名并弃用这个。

当检测器实现各种回调时，它们可以分析代码，如果它们发现有问题的模式，它们可以"报告"事件。这意味着计算错误消息，以及"位置"。事件的"位置"实际上是一个错误范围——一个文件，以及起始偏移和结束偏移。位置也可以链接在一起，所以例如对于"重复声明"错误，你可以并且应该包含两个位置。

许多检测器方法将传入一个`Context`，或者`Context`的更具体的子类，如`JavaContext`或`XmlContext`。这允许lint为检测器提供它们可能需要的信息，而无需传入大量参数。它还允许lint随时间添加其他数据而不破坏签名。

`Context`类还提供许多便利API。例如，对于`XmlContext`，有为XML标签、XML属性、XML属性的名称部分和XML属性的值部分创建位置的方法。对于`JavaContext`，还有创建位置的方法，如方法调用，包括是否包含接收器和/或参数列表。

当你报告`Incident`时，你也可以提供`LintFix`；这是IDE可以用来提供对警告采取行动的快速修复。在某些情况下，你可以提供完整和正确的修复（如删除未使用的元素）。在其他情况下，修复可能不太清楚；例如，`AccessibilityDetector`要求你为图像设置描述；快速修复将设置内容属性，但将文本值保留为TODO并选择字符串，以便用户只需键入即可替换它。

在报告事件时，确保错误消息不是通用的；尝试明确并包含当前场景的具体信息。例如，而不是只是"重复声明"，使用"`$name`已经被声明"。这不仅仅是为了美观；它还使lint的基线机制工作得更好，因为它目前通过id + 文件 + 消息匹配，而不是通过通常随时间漂移的行号。

### 客户端API与检测器API

Lint的API有两个部分：

* **客户端API**："从工具内集成（并运行）lint"。例如，IDE和构建系统都使用此API来嵌入和调用lint来分析项目或编辑器中的代码。
* **检测器API**："实现新的lint检查"。这是允许检查器分析代码并报告它们发现的问题的API。

客户端API中表示在工具中运行的lint的类称为`LintClient`。这个类负责，除了其他事情：

* **报告检测器发现的事件**。例如，在IDE中，它将在源编辑器中放置错误标记，在构建系统中，它可能将警告写入控制台或生成报告或甚至使构建失败。
* **处理I/O**。检测器永远不应该直接从磁盘读取文件。这允许lint检查在例如IDE中平滑工作。当lint即时运行时，lint检查请求源文件内容（或其他支持文件）时，IDE中的`LintClient`将实现`readFile`方法，首先在打开的源编辑器中查找，如果请求的文件正在编辑，它将返回当前（通常未保存！）内容。
* **处理网络流量**。Lint检查永远不应该自己打开URLConnections。相反，它们应该通过lint API请求URL的数据。除了其他事情，这允许`LintClient`使用配置的IDE代理设置（如在lint的IntelliJ集成中所做的）。这对测试也很好，因为`LintClient`的特殊单元测试实现有一种简单的方法为特定URL提供精确响应：

```kotlin
lint()
  .files(...)
  // 设置确切的预期maven.google.com网络输出以
  // 确保测试中版本建议的稳定性
  .networkData("https://maven.google.com/master-index.xml", ""
       + "<!--?xml version='1.0' encoding='UTF-8'?-->\n"
       + "<metadata>\n"
       + "  <com.android.tools.build>"
       + "</com.android.tools.build></metadata>")
  .networkData("https://maven.google.com/com/android/tools/build/group-index.xml", ""
       + "<!--?xml version='1.0' encoding='UTF-8'?-->\n"
       + "<com.android.tools.build>\n"
       + "  <gradle versions=\"2.3.3,3.0.0-alpha1\"/>\n"
       + "</gradle></com.android.tools.build>")
.run()
.expect(...)
```

以及更多、更多。**然而，`LintClient`的大部分实现是用于lint本身的集成，作为检查作者你不需要担心它。**检测器API将更重要，它也比客户端API更不可能改变。

两个部分之间的划分并不完美；一些类不能整齐地适合两者之间，或者历史上被放在错误的地方，所以这是一个需要注意但不是绝对的高级设计。

另外，

由于两个独立包之间的划分，回顾起来这是一个错误，许多仅用于内部lint使用的API已被设为`public`，以便lint在一个包中的代码可以从另一个包访问它。通常有注释解释这仅用于内部使用，但请注意，即使某些东西是`public`或不是`final`，调用或覆盖它可能也不是一个好主意。

### 创建Issue

有关如何设置项目并实际发布你的lint检查的信息，请参见示例和发布章节。

`Issue`是一个final类，所以与`Detector`不同，你不子类化它；你通过`Issue.create`实例化它。

按照惯例，问题在相应检测器的伴生对象内注册，但这不是必需的。

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
    ...
}
```

有许多事情需要注意。

在第4行，我们有`Issue.create()`调用。我们将问题存储到属性中，以便我们可以从`IssueRegistry`引用此问题，我们在其中向lint提供`Issue`，以及在`Detector`代码中报告问题的事件。

注意`Issue.create`是一个有很多参数的方法（我们可能会在将来添加更多参数）。因此，显式包含参数名称是一个好习惯（因此要在Kotlin中实现你的代码）。

`Issue`提供关于问题类型的元数据。

**`id`**是此问题的简短、唯一标识符。按照惯例，它是单词的组合，大写驼峰式（尽管你也可以添加自己的包前缀，如Java包）。注意id是"用户可见的"；它包含在lint在构建系统中运行时的文本输出中，如：

```
src/main/kotlin/test/pkg/MyTest.kt:4: Warning: Do not hardcode "/sdcard/";
      use Environment.getExternalStorageDirectory().getPath() instead [SdCardPath]
    val s: String = "/sdcard/mydir"
                     -------------
0 errors, 1 warnings
```

（注意错误消息末尾的`[SdCardPath]`后缀。）

id向用户公开的原因是ID是他们配置和/或抑制问题的方式。例如，要在当前方法中抑制警告，使用

```kotlin
@Suppress("SdCardPath")
```

（或在Java中，@SuppressWarnings）。注意有一个IDE快速修复来抑制事件，它将自动添加这些注解，所以你不需要知道ID就能够抑制事件，但ID将在它生成的注解中可见，所以它应该合理地具体。

另外，由于命名空间是全局的，尝试避免选择可能与其他冲突的通用名称，或者似乎涵盖比预期更大的问题集。例如，"InvalidDeclaration"将是一个糟糕的id，因为它可以涵盖跨多种语言和技术的声明的许多潜在问题。

接下来，我们有**`briefDescription`**。你可以将其视为"类别报告标题"；这是此类型所有事件的静态描述，因此不能包含任何具体信息。此字符串例如用作HTML报告中此类型所有事件的标题，在IDE中，如果你打开检查UI，各种问题使用简要描述在那里列出。

**`explanation`**是多行、理想情况下多段的问题说明。在某些情况下，问题是不言而喻的，如"未使用的声明"的情况，但在许多情况下，问题更微妙，可能需要额外的解释，特别是开发者应该**做**什么来解决问题。说明包含在HTML报告和IDE检查结果窗口中。

注意，即使我们使用原始字符串，即使字符串缩进以与问题注册的其余部分对齐以获得更好的可读性，我们不需要在原始字符串上调用`trimIndent()`。Lint自动执行此操作。

然而，我们确实需要添加行延续——这些是行末尾的尾随\\。

还要注意我们有类似Markdown的简单语法，在下面的"TextFormat"部分中描述。你可以使用星号表示斜体或双星号表示粗体，你可以使用撇号表示代码字体等。在终端输出中这没有区别，但IDE、解释、事件错误消息等，都使用这些样式格式化。

**`category`**不是超级重要的；主要用途是类别名称可以在问题配置时被视为id；例如，用户可以关闭所有国际化问题，或仅对安全相关问题运行lint。类别也用于在HTML报告中定位相关问题。如果没有合适的内置类别，你也可以创建自己的。

**`severity`**属性非常重要。问题可以是警告或错误。这些在IDE中被不同对待（错误是红色下划线，警告是黄色高亮），在构建系统中（错误可以选择性地破坏构建，警告不会）。还有一些其他严重性；"fatal"就像错误，除了这些检查被指定为足够重要（并且很少有误报），以至于我们在发布构建期间运行它们，即使用户没有明确运行lint目标。还有"informational"严重性，它只在一两个地方使用，最后是"ignore"严重性。这从来不是你为问题注册的严重性，但它是开发者可以为特定问题配置的严重性之一，从而关闭该特定检查。

你还可以指定**`moreInfo`** URL，它将作为"更多信息"链接包含在问题说明中，以打开阅读有关此问题或潜在问题的更多详细信息。

### TextFormat

lint中的所有错误消息和问题元数据字符串都使用简单的类似Markdown的语法进行解释：

| 原始文本格式 | 渲染为 |
|-------------|--------|
| This is a \`code symbol\` | This is a `code symbol` |
| This is \*italics\* | This is *italics* |
| This is \*\*bold\*\* | This is **bold** |
| This is \~\~strikethrough\~\~ | This is ~~strikethrough~~ |
| <http://>, <https://> | [http://](http://), [https://](https://) |
| \\\*not italics\* | \*not italics\* |
| \`\`\`language\\n text\\n\`\`\` | （预格式化文本块） |

这在Lint生成的HTML报告或IDE中显示错误消息和问题说明时很有用，例如错误消息工具提示将使用格式化。

在API中，有一个`TextFormat`枚举封装了不同的文本格式，上述语法被称为`TextFormat.RAW`；例如，它可以转换为`.TEXT`或`.HTML`，lint在将文本报告写入控制台或将HTML报告写入文件时分别执行此操作。作为lint检查作者，你不需要知道这个（尽管你可以例如使用单元测试支持决定你想要在预期输出中比较哪种格式），但这里的主要点是你的问题的简要描述、问题说明、事件报告消息等，应该使用上述"raw"语法。特别是第一个转换；错误消息经常引用类名和方法名，这些应该用撇号包围。

有关如何制作错误消息的更多信息，请参见错误消息章节。

### Issue实现

最后一个问题注册属性是**`implementation`**。这是我们将元数据粘合到可以找到此问题实例的分析器的特定实现的地方。

通常，`Implementation`提供两件事：

* 应该实例化的我们`Detector`的`.class`。在上面的代码示例中是`SdCardDetector`。
* 此问题的检测器适用的`Scope`。在上面的示例中是`Scope.JAVA_FILE`，这意味着它将应用于Java和Kotlin文件。

### 作用域

`Implementation`实际上接受一个作用域**集合**；我们仍然将其称为"作用域"。一些lint检查想要分析多种类型的文件。例如，`StringFormatDetector`将分析声明跨各种语言环境的格式字符串的资源文件，以及包含引用格式字符串的`String.format`调用的Java和Kotlin文件。

在`Scope`类中有许多预定义的作用域集合。`Scope.JAVA_FILE_SCOPE`是最常见的，它是一个包含确切`Scope.JAVA_FILE`的单例集合，但你总是可以创建自己的，例如

```kotlin
EnumSet.of(Scope.CLASS_FILE, Scope.JAVA_LIBRARIES)
```

当lint问题需要多个作用域时，这意味着lint将**仅**在**所有**作用域在运行工具中可用时运行此检测器。当lint运行完整的批处理运行（如Gradle lint目标或IDE中的完整"检查代码"）时，所有作用域都可用。

然而，当lint在编辑器中即时运行时，它只能访问当前文件；它不会为每几次击键重新分析项目中的_所有_文件。所以在这种情况下，lint驱动程序中的作用域只包括当前源文件的类型，只有指定作用域是子集的lint检查才会运行。

这是新lint检查作者的常见错误：lint检查在单元测试中工作得很好，但他们在IDE中看不到工作，因为问题实现请求多个作用域，**所有**必须可用。

通常，lint检查查看多种源文件类型以在所有情况下正确工作，但它仍然可以识别给定单个源文件的_一些_问题。在这种情况下，`Implementation`构造函数（接受作用域集合的vararg）可以传递额外的作用域集合，称为"分析作用域"。如果当前lint客户端的作用域匹配或是任何分析作用域的子集，则检查仍将运行。

### 注册Issue

一旦你创建了你的问题，你需要从`IssueRegistry`提供它。

这是一个示例`IssueRegistry`：

```kotlin
package com.example.lint.checks

import com.android.tools.lint.client.api.IssueRegistry
import com.android.tools.lint.client.api.Vendor
import com.android.tools.lint.detector.api.CURRENT_API

class SampleIssueRegistry : IssueRegistry() {
    override val issues = listOf(SdCardDetector.ISSUE)

    override val api: Int
        get() = CURRENT_API

    // works with Studio 4.1 or later; see
    // com.android.tools.lint.detector.api.Api / ApiKt
    override val minApi: Int
        get() = 8

    // Requires lint API 30.0+; if you're still building for something
    // older, just remove this property.
    override val vendor: Vendor = Vendor(
        vendorName = "Android Open Source Project",
        feedbackUrl = "https://com.example.lint.blah.blah",
        contact = "author@com.example.lint"
    )
}
```

在第8行，我们返回我们的问题。这是一个列表，所以`IssueRegistry`可以提供多个问题。

**`api`**属性应该完全像它在你自己的问题注册表中出现的方式一样编写；这将记录此问题注册表编译时使用的lint API版本（因为这引用一个静态最终常量，它将被复制到jar文件中，而不是在加载jar时动态查找）。

**`minApi`**属性记录此检查已测试的最旧lint API级别。

这两个都在问题加载时用于确保lint检查兼容，但在最近版本的lint（7.0）中，lint将更积极地尝试加载较旧的检测器，即使它们已经针对较旧的API编译，因为它们工作的可能性很高（它检查字节码中的所有lint API并使用反射验证它们仍然存在）。

**`vendor`**属性从7.0开始是新的，给lint作者一种指示lint检查来源的方式。当用户使用lint时，他们运行数百个检查，有时不清楚联系谁提出请求或错误报告。当指定供应商时，lint将在错误输出和报告中包含此信息。

使lint检查可用的最后一步是通过服务加载器机制使`IssueRegistry`知晓。

创建一个名为

```
src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
```

的文件，内容如下（但你要替换为你自己的问题注册表的完全限定类名）：

```
com.example.lint.checks.SampleIssueRegistry
```

如果你不使用Gradle构建你的lint检查，你可能不想要`src/main/resources`前缀；重点是你的jar文件打包应该在jar文件的根目录包含`META-INF/services/`。

### 实现检测器：扫描器

我们终于来到编写lint检查的主要任务：实现**`Detector`**。

这是一个简单的：

```kotlin
class MyDetector : Detector() {
   override fun run(context: Context) {
       context.report(ISSUE, Location.create(context.file),
           "I complain a lot")
   }
}
```

这将在每个文件中抱怨。显然，没有真正的lint检测器这样做；我们想要做一些分析并**有条件地**报告事件。有关如何措辞错误消息的信息，请参见错误消息章节。

为了使分析更简单，Lint对分析各种文件类型有专门的支持。工作方式是你注册兴趣，然后将调用各种回调。

例如：

* 当实现**`XmlScanner`**时，在XML元素中你可以被回调
  * 当声明任何给定标签集合时（`visitElement`）
  * 当声明任何命名属性集合时（`visitAttribute`）
  * 你可以通过`visitDocument`执行自己的文档遍历
* 当实现**`SourceCodeScanner`**时，在Kotlin和Java文件中你可以被回调
  * 当调用给定名称的方法时（`getApplicableMethodNames`和`visitMethodCall`）
  * 当实例化给定类型的类时（`getApplicableConstructorTypes`和`visitConstructor`）
  * 当声明扩展（可能间接）给定类或接口的新类时（`applicableSuperClasses`和`visitClass`）
  * 当引用或组合注解元素时（`applicableAnnotations`和`visitAnnotationUsage`）
  * 当出现给定类型的任何AST节点时（`getApplicableUastTypes`和`createUastHandler`）
* 当实现**`ClassScanner`**时，在`.class`和`.jar`文件中你可以被回调
  * 当为特定所有者调用方法时（`getApplicableCallOwners`和`checkCall`）
  * 当发生给定字节码指令时（`getApplicableAsmNodeTypes`和`checkInstruction`）
  * 像XmlScanner的`visitDocument`一样，你可以通过`checkClass`执行自己的ASM字节码迭代
* 还有各种其他扫描器，例如`GradleScanner`允许你访问`build.gradle`和`build.gradle.kts` DSL闭包，`BinaryFileScanner`访问资源文件如webp和png文件，`OtherFileScanner`允许你访问未知文件。

注意`Detector`已经为所有这些接口实现了空的存根方法，所以如果你例如在检测器中实现`SourceFileScanner`，你不需要去添加所有你不使用的方法的空实现。

Lint的API都不要求你在覆盖方法时调用`super`；意图被覆盖的方法总是空的，所以super调用是多余的。

### 检测器生命周期

检测器注册是按检测器类完成的，而不是按检测器实例。Lint将代表你实例化检测器。它将为每个分析实例化检测器一次，所以你可以在字段中存储检测器上的状态并累积信息以在最后进行分析。

在分析每个单独文件之前和之后都有一些回调（`beforeCheckFile`和`afterCheckFile`），以及在分析所有模块之前和之后（`beforeCheckRootProject`和`afterCheckRootProject`）。

这例如是"未使用资源"检查的工作方式：当我们处理每个文件时，我们存储在项目中找到的所有资源声明和资源引用，然后在`afterCheckRootProject`方法中我们分析资源图并计算在引用图中不可达的任何资源声明，然后我们将每个这些报告为未使用。

### 扫描器顺序

一些lint检查涉及多个扫描器。这在Android中很常见，我们想要交叉检查资源文件中的数据与代码使用的一致性。例如，`String.format`检查确保传递给`String.format`的参数与所有翻译XML文件中指定的格式字符串匹配。

Lint定义了处理扫描器的确切顺序，在扫描器内，数据。这使得编写一些检测器更容易，因为你知道你会在另一个之前遇到一种类型的数据；你不必处理相反的顺序。例如，在我们的`String.format`示例中，我们知道我们总是会在看到带有`String.format`调用的代码之前看到格式字符串，所以我们可以将格式字符串存储在映射中，当我们处理代码中的格式调用时，我们可以立即发出报告；我们不必担心遇到我们尚未处理的格式字符串的格式调用。

这是lint定义的顺序：

1. Android Manifest
2. Android资源XML文件（按文件夹类型字母顺序，所以例如布局在值文件如翻译之前处理）
3. Kotlin和Java文件
4. 字节码（本地`.class`文件和库`.jar`文件）
5. TOML文件
6. Gradle文件
7. 其他文件
8. ProGuard文件
9. 属性文件

类似地，lint总是会在依赖它们的模块之前处理库。

如果你需要访问迭代顺序后面的内容，并且存储所有当前数据并在遇到后面的数据时处理它不实际，注意lint支持"多遍分析"：它可以多次运行数据。调用此功能的方式是通过`context.driver.requestRepeat(this, …)`。这实际上是未使用资源分析的工作方式。但是请注意，此重复仅在当前模块内有效；你不能通过整个依赖图重新运行分析。

### 实现检测器：服务

除了扫描器，lint还提供许多服务以使实现更简单。这些包括

* **`ConstantEvaluator`**：执行AST表达式的求值，所以例如如果我们有语句`x = 5; y = 2 * x`，常量求值器可以告诉你y是10。这个常量求值器也可以比编译器的严格常量求值器更宽松；例如它可以返回不是所有部分都已知的连接字符串，或者它可以使用字段的非final初始值。这可以帮助你找到_可能_的错误而不是_确定_的错误。
* **`TypeEvaluator`**：尝试提供表达式的具体类型。例如，对于Java语句`Object s = new StringBuilder(); Object o = s`，类型求值器可以告诉你`o`此时的类型实际上是`StringBuilder`。
* **`JavaEvaluator`**：尽管名称不幸较旧，此服务适用于Kotlin和Java，例如可以提供关于继承层次结构、从完全限定名称进行类查找等的信息。
* **`DataFlowAnalyzer`**：方法内的数据流分析。
* 对于Android分析，有几个其他重要的服务，如`ResourceRepository`和`ResourceEvaluator`。
* 最后，有许多实用方法；例如有一个用于查找可能拼写错误的`editDistance`方法。

### 扫描器示例

让我们使用上述扫描器之一`XmlScanner`创建一个`Detector`，它将查看项目中的所有XML文件，如果它遇到`<bitmap>`标签，它将报告应该使用`<vector>`：

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

上述使用Lint 7.0及以后的新`Incident` API；在较旧版本中你可以使用以下API，它在7.0中仍然工作：

```kotlin
class MyDetector : Detector(), XmlScanner {
    override fun getApplicableElements() = listOf("bitmap")

    override fun visitElement(context: XmlContext, element: Element) {
        context.report(ISSUE, context.getLocation(element),
            "Use `<vector>` instead of `<bitmap>`")
    }
}
```

第二种（较旧的）形式可能看起来更简单，但新API允许将更多元数据附加到报告，如覆盖严重性。你不必转换为构建器语法来做到这一点；你也可以将第二种形式写为

```kotlin
context.report(Incident(ISSUE, context.getLocation(element),
    "Use `<vector>` instead of `<bitmap>`"))
```

### 分析Kotlin和Java代码

#### UAST

要分析Kotlin和Java代码，lint提供代码的抽象语法树或"AST"。

这个AST称为"UAST"，代表"通用抽象语法树"，它以相同的方式表示多种语言，隐藏语言特定的细节，如语句末尾是否有分号或注解类的声明方式是`@interface`还是`annotation class`等。

这使得可能编写一个在UAST支持的所有语言中工作的单一分析器。这非常有用；大多数lint检查都在做API或数据流特定的事情，而不是语言特定的事情。如果你确实需要实现非常语言特定的东西，请参见下一节"PSI"。

在UAST中，每个元素称为**`UElement`**，有许多子类——`UFile`用于编译单元，`UClass`用于类，`UMethod`用于方法，`UExpression`用于表达式，`UIfExpression`用于`if`表达式等。

这是两个用Kotlin和Java编写的等效程序的UAST中AST的可视化。这些程序都产生右侧显示的相同AST：一个`UFile`编译单元，包含一个名为`MyTest`的`UClass`，包含一个名为s的`UField`，它有一个初始化器将初始值设置为`hello`。

名称"UAST"有点误导；它不是所有可能语法树的某种超集；相反，将其视为所有代码的"Java视图"。所以，例如，没有表示Kotlin属性的`UProperty`节点。相反，AST看起来与属性在Java中实现的相同：它将包含一个私有字段和一个公共getter和一个公共setter（当然，除非Kotlin属性指定私有setter）。如果你在Kotlin中编写代码并尝试从Java文件访问该Kotlin代码，你会看到同样的事情——Kotlin的"Java视图"。下一节"PSI"将讨论如何进行更多语言特定的分析。

#### UAST示例

这是一个示例（来自内置的Android `AlarmDetector`），显示了上述所有实践；这是一个lint检查，确保如果任何人调用`AlarmManager.setRepeating`，第二个参数至少是5,000，第三个参数至少是60,000。

第1行说我们想要在lint遇到对`setRepeating`的方法时调用第3行。

在第8-14行，我们确保我们谈论的是正确类上的正确方法，具有正确的签名。这使用`JavaEvaluator`检查被调用的方法是命名类的成员。这是必要的，因为如果lint遇到像`Unrelated.setRepeating`这样的方法调用，回调也会被调用；`visitMethodCall`回调只按名称匹配，不按接收器。

在第36行，我们使用`ConstantEvaluator`计算传入的每个参数的值。这将让这个lint检查不仅处理你直接在参数列表中指定特定值的情况，还处理例如从其他地方引用常量的情况。

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

private fun ensureAtLeast(
    context: JavaContext,
    node: UCallExpression,
    parameter: Int,
    min: Long
) {
    val argument = node.valueArguments[parameter]
    val value = getLongValue(context, argument)
    if (value < min) {
        val message = "Value will be forced up to $min as of Android 5.1; " +
            "don't rely on this to be exact"
        context.report(ISSUE, argument, context.getLocation(argument), message)
    }
}

private fun getLongValue(
    context: JavaContext,
    argument: UExpression
): Long {
    val value = ConstantEvaluator.evaluate(context, argument)
    if (value is Number) {
        return value.toLong()
    }

    return java.lang.Long.MAX_VALUE
}
```

#### 查找UAST

要编写检测器的分析，你需要知道你感兴趣的代码的AST是什么样子的。而不是尝试通过在调试器下检查元素来弄清楚，一个简单的方法是"美化打印"它，使用`UElement`扩展方法**`asRecursiveLogString`**。

例如，给定以下单元测试：

```kotlin
lint().files(
       kotlin(""
               + "package test.pkg\n"
               + "\n"
               + "class MyTest {\n"
               + "    val s: String = \"hello\"\n"
               + "}\n"), ...
```

如果你从其中一个回调中计算`context.uastFile?.asRecursiveLogString()`，它将打印：

```
UFile (package = test.pkg)
    UClass (name = MyTest)
        UField (name = s)
            UAnnotation (fqName = org.jetbrains.annotations.NotNull)
            ULiteralExpression (value = "hello")
        UAnnotationMethod (name = getS)
        UAnnotationMethod (name = MyTest)
```

（这也说明了关于UAST表示代码Java视图的早期观点；这里只读公共Kotlin属性"s"由私有字段`s`和公共getter方法`getS()`表示。）

#### 解析

当你有方法调用或字段引用时，你可能想要查看被调用的方法或字段。这称为"解析"，UAST直接支持它；在`UCallExpression`上例如调用`.resolve()`，它返回`PsiMethod`，它像`UMethod`，但可能不表示我们有源的方法（例如如果你解析对JDK或我们没有源的库的引用，这将是情况）。如果源可用，你可以在PSI元素上调用`.toUElement()`尝试将其转换为UAST。

解析仅在lint具有正确的类路径以便引用的方法、字段或类实际存在时才有效。如果不是，解析将返回null，各种lint回调将不会被调用。这是lint检查"不工作"的常见问题来源；它经常出现在lint单元测试中，测试文件将引用实际上不包含在类路径中的某些API。对此的推荐方法是声明本地存根。有关此的更多详细信息，请参见单元测试章节。

#### 隐式调用

Kotlin支持许多内置操作符的操作符重载。例如，如果你有以下代码，

```kotlin
fun test(n1: BigDecimal, n2: BigDecimal) {
    // Here, this is really an infix call to BigDecimal#compareTo
    if (n1 < n2) {
        ...
    }
}
```

这里的`<`实际上是一个函数调用（你可以通过在IDE中对符号调用Go To Declaration来验证）。这不是专门为`BigDecimal`类构建的；如果你在函数声明中放置`operator`修饰符，这也适用于你的任何Java类，以及Kotlin。

然而，注意在抽象语法树中，这**不是**表示为`UCallExpression`；这里我们将有一个`UBinaryExpression`，左操作数`n1`，右操作数`n2`和操作符`UastBinaryOperator.LESS`。这意味着如果你的lint检查专门查看`compareTo`调用，你不能只访问每个`UCallExpression`；你_也_必须访问每个`UBinaryExpression`，并检查它是否调用`compareTo`方法。

这不仅适用于二元操作符；它也适用于一元操作符（如`!`、`-`、`++`等），以及甚至数组访问；数组访问可以映射到`get`调用或`set`调用，取决于如何使用。

Lint有一些特殊支持来帮助处理这些情况。

首先，对调用回调的内置支持（你通过从`getApplicableMethodNames`返回名称注册对调用名称的兴趣，然后在`visitMethodCall`回调中响应）已经自动处理这个。如果你例如注册对方法调用`compareTo`的兴趣，它也会为上面显示的二元操作符场景调用你的回调，传递给你一个具有正确值参数、方法名称等的调用。

工作方式是lint可以创建一个"包装器"类，将底层的`UBinaryExpression`（或`UArrayAccessExpression`等）呈现为`UCallExpression`。在二元操作符的情况下，值参数列表将是左和右操作数。这意味着你的代码可以只处理这个，就像代码写成显式调用而不是使用操作符语法一样。你也可以直接寻找这个包装器类`UImplicitCallExpression`，它有一个用于查找原始或底层元素的访问器方法。你可以通过`UBinaryExpression.asCall()`、`UUnaryExpression.asCall()`和`UArrayAccessExpression.asCall()`自己构造这些包装器。

还有一个你可以使用的访问者来访问所有调用——`UastCallVisitor`，它将访问所有调用，包括来自数组访问和一元操作符和二元操作符的调用。

这个支持对数组访问特别有用，因为与操作符表达式不同，`UArrayExpression`上没有`resolveOperator`方法。UAST问题跟踪器中有一个开放请求（KTIJ-18765），但现在，lint有一个解决方法来自己处理解析。

#### PSI

PSI是"程序结构接口"的缩写，是IntelliJ的AST抽象，用于IDE中的所有语言建模。

注意每种语言都有**不同的**PSI表示。Java和Kotlin涉及完全不同的PSI类。这意味着使用PSI编写lint检查将涉及编写大量逻辑两次；一次用于Java，一次用于Kotlin。（Kotlin PSI处理起来有点棘手。）

这就是UAST的用途：有一个从Java PSI到UAST的"桥接"，有一个从Kotlin PSI到UAST的桥接，你的lint检查只分析UAST。

然而，有一些场景我们必须使用PSI。

第一个，也是最常见的，在前面关于解析的部分中列出。UAST不完全替代PSI；事实上，PSI在部分UAST API表面中泄漏。例如，`UMethod.resolve()`返回`PsiMethod`。更重要的是，`UMethod` **扩展** `PsiMethod`。

由于历史原因，`PsiMethod`和其他PSI类包含一些只对Java有效的不幸API，如请求方法体。因为`UMethod`扩展`PsiMethod`，你可能会试图在其上调用`getBody()`，但这将从Kotlin返回null。如果你的lint检查的单元测试只有用Java编写的测试案例，你可能没有意识到你的检查做错了事情，不会在Kotlin代码上工作。它应该在`UMethod`上调用`uastBody`。Lint的lint检测器特殊检测器寻找这个和其他一些场景（如调用`parent`而不是`uastParent`），所以确保为你的项目配置它。

当你处理"签名"——查看类和类继承、方法、参数等——使用PSI是可以的——也是不可避免的，因为UAST不表示字节码（尽管在将来它可能通过反编译器）或除了Kotlin和Java之外的任何其他JVM语言。

然而，如果你查看方法或类或字段初始化器_内部_的任何内容，你**必须**使用UAST。

你可能需要使用PSI的**第二个**场景是你必须做一些UAST中没有表示的语言特定的事情。例如，如果你试图查找参数的名称或默认值，或者给定类是否是伴生对象，那么你需要深入Kotlin PSI。

通常没有必要查看Java PSI，因为UAST完全涵盖了它，除非你想查看单独的细节，如AST节点之间的特定空格，这在PSI中表示但不在UAST中。

你可以从JetBrains找到PSI和UAST的其他文档。只是注意他们的文档针对IDE插件开发者而不是lint开发者。

### 测试

为lint检查编写单元测试很重要，这在专门的单元测试章节中详细介绍。
