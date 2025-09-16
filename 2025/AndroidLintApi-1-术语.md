# Android Lint API 指南 - 第1章：术语


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第1章节"Terminology"的中文翻译版本。

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
`Implementation`告诉lint如何实际分析给定的问题，例如要实例化哪个检测器类，以及检测器适用于哪些作用域。

### Incident（事件）
在特定位置发生的问题的特定实例。事件的示例是：

```
Warning: In file IoUtils.kt, line 140, the field download folder
is "/sdcard/downloads"; do not hardcode the path to `/sdcard`.
```

### Issue（问题）
你的lint检查识别的问题类型或类别。问题具有关联的严重性（错误、警告或信息）、优先级、类别、说明等。

问题的示例是"不要硬编码路径到/sdcard"。

### IssueRegistry（问题注册表）
`IssueRegistry`向lint提供问题列表。当你编写一个或多个lint检查时，你将在`IssueRegistry`中注册这些检查，并使用`META-INF`服务加载器机制指向它。

### LintClient（Lint客户端）
`LintClient`表示检测器运行的特定工具。例如，在IDE中运行时有一个LintClient，它（当报告事件时）会在编辑器中显示高亮，而当lint作为Gradle插件的一部分运行时，事件会被累积到HTML（和XML和文本）报告中，并在出错时中断构建。

### Location（位置）
"位置"指的是报告事件的地方。通常这指的是源文件中的文本范围，但位置也可以指向二进制文件，如`png`文件。位置也可以链接在一起，以及描述。因此，如果你例如报告重复声明，你可以包括**两个**位置，在IDE中，两个位置（如果它们在同一文件中）都会被高亮显示。从另一个链接的位置称为"次要"位置，但链接可以任意长（lint的单元测试基础设施将确保没有循环）。

### Partial Analysis（部分分析）
lint中的"map reduce"架构，使得可以单独分析各个模块，然后基于这些模块外部的条件过滤和自定义部分结果。这在部分分析章节中有更详细的解释。

### Platform（平台）
`Platform`抽象允许lint问题指示它们适用的地方（如"Android"或"Server"等）。这意味着Android特定的检查不会在非Android代码上触发警告。

### Scanner（扫描器）
`Scanner`是检测器可以实现的特定接口，用于指示它支持特定的回调集合。例如，`XmlScanner`接口定义了访问XML元素和属性的方法，`ClassScanner`定义了ASM字节码处理方法等。

### Scope（作用域）
`Scope`是一个枚举，列出了检测器可能想要分析的各种类型的文件。

例如，有XML文件的作用域，有Java和Kotlin文件的作用域，有.class文件的作用域等。

通常lint关心的是哪个**作用域集合**适用，所以大多数API采用`EnumSet<Scope>`，但我们通常将其称为"作用域"而不是"作用域集合"。

### Severity（严重性）
对于问题，事件应该是错误，还是只是警告，或者都不是（只是FYI高亮）。还有一种特殊类型的错误严重性，"fatal"，稍后讨论。

### TextFormat（文本格式）
描述lint理解的各种文本格式的枚举。Lint检查通常只使用"raw"格式，这是类似markdown的格式（例如，你可以用星号包围单词使其斜体，或用两个星号使其粗体等）。

### Vendor（供应商）
`Vendor`是一个简单的数据类，提供关于lint检查来源的信息：谁编写了它，在哪里提交问题等。
