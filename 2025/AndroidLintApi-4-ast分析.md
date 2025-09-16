# Android Lint API 指南 - 第4章：AST分析


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第4章节"AST Analysis"的中文翻译版本。

## AST分析

要分析Kotlin和Java文件，lint提供许多便利回调，使完成常见任务变得简单：

* 检查对特定方法名的调用
* 实例化特定类
* 扩展特定超类或接口
* 使用特定注解，或调用注解方法

以及更多。有关更多信息，请参见`SourceCodeScanner`接口。

它还有各种助手，如`ConstantEvaluator`和`DataFlowAnalyzer`来帮助分析代码。

但在某些情况下，你需要深入并自己分析"AST"。

### AST分析

AST是"抽象语法树"的缩写——源代码的树表示。考虑以下非常简单的Java程序：

```java
// MyTest.java
package test.pkg;

public class MyTest {
     String s = "hello";
}
```

这是上述程序AST的表示，它在IntelliJ中内部表示的方式。

这实际上是一个简化的视图；实际上，还有空白节点跟踪这些节点之间的所有空白字符跨度。

无论如何，你可以看到这里有相当多的细节——跟踪关键字、变量、对包的引用等——以及更高级的概念，如类和字段，我用较粗的边框标记了。

这是相应的Kotlin程序：

```kotlin
// MyTest.kt
package test.pkg

class MyTest {
    val s: String = "hello"
}
```

这是IntelliJ中相应的AST：

这个程序等价于Java程序。但注意它有完全不同的形状！它们引用不同的元素类，`PsiClass`对`KtClass`，一直向下。

但有一些共同点——它们每个都有文件的节点，类的节点，字段的节点，以及初始值，字符串。

### UAST

我们可以构造一个表示相同概念的新AST：

这是一个统一的AST，在称为"UAST"的东西中，"统一抽象语法树"的缩写。UAST是我们在Lint中用于代码的主要AST表示。这里所有的节点类都以大写U为前缀，代表UAST。这是上面第一个Java文件示例的UAST。

这是相应Kotlin示例的UAST：

如你所见，AST并不总是相同的。对于字符串，在Kotlin中，我们经常得到一个额外的父`UInjectionHost`。但对于我们的目的，你可以看到AST大部分是相同的，所以如果你处理Kotlin场景，你也会处理Java场景。

### UAST：Java视图

注意名称中的"统一"在这里有点误导。从名称你可能假设这是跨语言的AST的某种超集——一个可以表示所有语言所需的一切的AST。但情况并非如此！相反，更好的思考方式是这是AST的**Java视图**。

如果你例如有以下Kotlin数据类：

```kotlin
data class Person(
    var id: String,
    var name: String
)
```

这是一个有两个属性的Kotlin数据类。所以你可能期望UAST会有一种表示这些概念的方式。这应该是一个有两个`UProperty`子项的`UDataClass`，对吧？

但Java不支持属性。如果你尝试从Java访问`Person`实例，你会注意到它暴露了许多你在Kotlin代码中看不到的公共方法——除了`getId`、`setId`、`getName`和`setName`，还有`component1`和`component2`（用于解构），以及`copy`。

这些方法可以直接从Java调用，所以它们在UAST中显示，你的分析可以推理它们。

考虑另一个完整的Kotlin源文件，`test.kt`：

```kotlin
var property = 0
```

这是UAST表示：

这里我们有一个非常简单的Kotlin文件——单个Kotlin属性。但注意在UAST级别，没有顶级方法和属性这样的东西。在Java中，一切都是类，所以`kotlinc`会创建一个"外观类"，使用文件名加"Kt"。所以我们看到我们的`TestKt`类。这里有三个成员。有这个属性的getter和setter，作为`getProperty`和`setProperty`。然后有私有字段本身，属性存储的地方。

这一切都在UAST中显示。这是Kotlin代码的Java视图。这可能看起来有限制，但实际上，对于大多数lint检查，这实际上是你想要的。这使得推理API调用等变得容易。

### 表达式

你可能得到的印象是UAST树非常浅，只表示高级声明，如文件、类、方法和属性。

情况并非如此。虽然它**确实**跳过低级、语言特定的细节，如空白节点和单独的关键字节点，但所有各种表达式类型都被表示并可以推理。取以下表达式：

```kotlin
if (s.length > 3) 0 else s.count { it.isUpperCase() }
```

这映射到以下UAST树：

如你所见，它建模if、比较、lambda等。

### UElement

UAST中的每个节点都是`UElement`的子类。有一个父指针，这对于在AST中导航很方便。

你需要编写lint检查的真正技能是理解AST，然后对其进行模式匹配。一个简单的技巧是创建你想要的Kotlin或Java代码，在单元测试中，然后在你的检测器中，递归打印UAST作为树。

或者在调试器中，任何时候你有一个`UElement`，你可以调用`UElement.asRecursiveLogString`，评估并看看你发现什么。

例如，对于以下Kotlin代码：

```kotlin
import java.util.Date
fun test() {
    val warn1 = Date()
    val ok = Date(0L)
}
```

这是相应的UAST `asRecursiveLogString`输出：

```
UFile (package = )
  UImportStatement (isOnDemand = false)
  UClass (name = JavaTest)
    UMethod (name = test)
      UBlockExpression
        UDeclarationsExpression
          ULocalVariable (name = warn1)
            UCallExpression (kind = UastCallKind(name='constructor_call'), …
              USimpleNameReferenceExpression (identifier = Date)
        UDeclarationsExpression
          ULocalVariable (name = ok)
            UCallExpression (kind = UastCallKind(name='constructor_call'), …
              USimpleNameReferenceExpression (identifier = Date)
              ULiteralExpression (value = 0)
```

### 访问

你通常不应该自己访问源文件。Lint有一个特殊的`UElementHandler`，它用于确保我们不会重复访问源文件数千次，每个检测器一次。

但当你进行本地分析时，你有时需要访问子树。

要做到这一点，只需扩展`AbstractUastVisitor`并将访问者传递给相应`UElement`的`accept`方法。

```kotlin
method.accept(object : AbstractUastVisitor() {
    override fun visitSimpleNameReferenceExpression(node: USimpleNameReferenceExpression): Boolean {
        // your code here
        return super.visitSimpleNameReferenceExpression(node)
    }
})
```

在访问者中，你通常想要调用`super`，如上所示。如果你已经"看够了"并可以停止访问AST的其余部分，你也可以`return true`。

如果你访问Java PSI元素，你使用`JavaRecursiveElementVisitor`，在Kotlin PSI中，使用`KtTreeVisitor`。

### UElement到PSI映射

UAST构建在PSI之上，每个`UElement`都有一个`sourcePsi`属性（可能为null）。这让你从通用UAST节点映射到特定PSI元素。

这是一个说明：

我们在右上角有我们的UAST树。这里是幕后的Java PSI AST。我们可以通过访问`sourcePsi`属性访问`UElement`的底层PSI节点。所以当你确实需要深入语言特定的东西时，这很简单。

注意在某些情况下，这些引用为null。

大多数`UElement`节点指向PSI AST——无论是Java AST还是Kotlin AST。这里是相同的AST，但为每个节点添加了`sourcePsi`属性的**类型**。

你可以看到生成的包含顶级函数的外观类有一个null `sourcePsi`，因为在Kotlin PSI中，没有外观类的真实`KtClass`。对于三个成员，私有字段和getter和setter，它们都对应于完全相同的单个`KtProperty`实例，Kotlin PSI中生成这些方法的单个节点。

### PSI到UElement

在某些情况下，我们也可以使用`toUElement`扩展函数从PSI元素映射回UAST。

例如，假设我们解析方法调用。这返回`PsiMethod`，而不是`UMethod`。但我们可以使用以下方法获得相应的`UMethod`：

```kotlin
val resolved = call.resolve() ?: return
val uDeclaration = resolve.toUElement()
```

但是注意`toUElement`可能返回null。例如，如果你解析到编译的方法调用（你可以使用`resolved is PsiCompiledElement`检查），UAST无法转换它。

### UAST与PSI

UAST是编写Kotlin和Java的lint检查时使用的首选AST。它让你推理跨语言相同的事情。声明。函数调用。超类。赋值。If表达式。Return语句。等等。

有些lint检查是语言特定的——例如，如果你编写一个禁止使用伴生对象的lint检查——在这种情况下，使用UAST而不是PSI没有大的优势；它只会在Kotlin代码上运行。（但是注意lint的API和便利回调都针对UAST，所以即使对于语言特定的检查，编写UAST lint检查也更容易。）

然而，绝大多数lint检查不是语言特定的，它们是**API**或错误模式特定的。如果API可以从Java调用，你希望你的lint检查不仅标记Kotlin中的问题，也标记Java代码中的问题。你不想写两次lint检查——所以如果你使用UAST，单个lint检查可以对两者都工作。但虽然你通常想要使用UAST进行分析（lint的API通常围绕UAST定向），但有些情况下使用PSI是合适的。

特别是，当你做高度语言特定的事情，以及UAST中没有暴露语言细节的地方，你应该使用PSI。

例如，假设你需要确定`UClass`是否是Kotlin"伴生对象"。你可以作弊并查看类名是否为"Companion"。但这不完全正确；在Kotlin中你可以指定自定义伴生对象名称，当然用户可以自由创建名为"Companion"但不是伴生对象的类：

```kotlin
class Test {
    companion object MyName { // 伴生对象不叫"Companion"！
    }

    object Companion { // 叫"Companion"但不是伴生对象！
    }
}
```

正确的方法是使用Kotlin PSI，通过`UElement.sourcePsi`属性：

```kotlin
// Skip companion objects
val source = node.sourcePsi
if (source is KtObjectDeclaration && source.isCompanion()) {
    return
}
```

（要弄清楚如何编写上述代码，在测试案例上使用调试器并查看`UClass.sourcePsi`属性；你会发现它是`KtObjectDeclaration`的某个子类；查找其最一般的超接口或类，然后使用代码完成发现可用的API，如`isCompanion()`。）

### Kotlin分析API

使用Kotlin PSI是正确分析Kotlin代码直到最近的技术状态。但当你查看PSI时，你会发现有些事情真的很难完成——特别是解析引用，以及处理Kotlin类型。

Lint实际上没有给你访问如果你想要尝试在Kotlin PSI中查找类型所需的一切；你需要称为"绑定上下文"的东西，它没有在任何地方暴露！这个遗漏是故意的，因为这是旧编译器的实现细节。未来是K2；编译器前端的完全重写，不再使用旧的绑定上下文。作为K2工具支持的一部分，有一个新的API叫做"Kotlin分析API"，你可以用它来深入了解Kotlin的细节。

对于大多数lint检查，如果可以的话，你应该只使用UAST。但当你需要知道真正**详细的**Kotlin信息时，特别是围绕类型、智能转换和null推断等，Kotlin分析API是你最好的朋友（也是唯一的选择...）

Kotlin分析API还不是最终的，可能会继续变化。事实上，大多数符号最近被重命名了。例如，`analyze`返回的`KtAnalysisSession`已被重命名为`KaSession`。大多数API现在有前缀`Ka`。

#### Nothing类型？

这是一个简单的例子：

```kotlin
fun testTodo() {
   if (SDK_INT < 11) {
       TODO() // never returns
   }
   val actionBar = getActionBar() // OK - SDK_INT must be >= 11 !
}
```

这里我们有一个场景，我们知道TODO调用永远不会返回，lint可以在分析控制流时利用这一点——特别是，它应该理解在TODO()调用后没有穿透的机会，所以它可以得出结论SDK_INT在if块后必须至少是11。

Kotlin编译器能够推理这一点的方式是标准库中的`TODO`方法有`Nothing`的返回类型。

```kotlin
@kotlin.internal.InlineOnly
public inline fun TODO(): Nothing = throw NotImplementedError()
```

`Nothing`返回类型意味着它永远不会返回。

在Kotlin lint分析API之前，lint没有推理`Nothing`类型的方法。UAST只返回Java类型，它映射到void。所以相反，lint有一个丑陋的hack，只是硬编码不返回的方法的知名名称：

```kotlin
if (nextStatement is UCallExpression) {
    val methodName = nextStatement.methodName
    if (methodName == "fail" || methodName == "error" || methodName == "TODO") {
        return true
    }
```

然而，使用Kotlin分析API，这很容易：

```kotlin
fun callNeverReturns(call: UCallExpression): Boolean {
    val sourcePsi = call.sourcePsi as? KtCallExpression ?: return false
    analyze(sourcePsi) {
        val callInfo = sourcePsi.resolveToCall() ?: return false
        val returnType = callInfo.singleFunctionCallOrNull()?.symbol?.returnType
        return returnType != null && returnType.isNothingType
    }
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
/**
 * Returns true if this [call] node calls a method known to never
 * return, such as Kotlin's standard library method "error".
 */
fun callNeverReturns(call: UCallExpression): Boolean {
    val sourcePsi = call.sourcePsi as? KtCallExpression ?: return false
    analyze(sourcePsi) {
        val callInfo = sourcePsi.resolveCall() ?: return false
        val returnType = callInfo.singleFunctionCallOrNull()?.symbol?.returnType
        return returnType != null && returnType.isNothing
    }
}
```

所有Kotlin分析API用法的入口点是调用`analyze`方法（见第7行）并传入Kotlin PSI元素。这创建了一个"分析会话"。**非常重要的是你不要将会话内的对象泄漏到会话外**——以避免内存泄漏和其他问题。如果你确实需要保持符号并稍后比较，你可以创建一个特殊的符号指针。

无论如何，有大量扩展方法将分析会话作为接收者，所以在第7到13行的lambda内，有许多新方法可用。

这里，我们有一个`KtCallExpression`，在`analyze`块内我们可以调用`resolveCall()`来到达被调用方法的符号。

类似地，在`KtDeclaration`（如命名函数或属性）上我可以调用`.symbol`来获得该方法或属性的符号，例如查找参数信息。在`KtExpression`（如if语句）上我可以调用`.expressionType`来获得Kotlin类型。

`KaSymbol`和`KaType`是我们在Kotlin分析API中使用的基本原语。有许多符号的子类，如`KaFileSymbol`、`KaFunctionSymbol`、`KaClassSymbol`等。

在`callNeverReturns`的新实现中，我们解析调用，查找相应的函数，当然它本身是`KaSymbol`，从中我们获得返回类型，然后我们可以只检查它是否是`Nothing`类型。

这个API对旧Kotlin编译器（现在在lint中使用）和K2都有效，可以通过标志打开，很快将成为默认（当你读到这个时可能已经是默认了；我们不总是记得更新文档...）

#### 编译元数据

访问UAST中不可用的Kotlin特定知识是分析API的一个用途。

分析API的另一个大优势是它让你访问推理编译的Kotlin代码，就像编译器一样。

通常，当你用UAST解析时，你只是得到一个普通的`PsiMethod`。例如，如果我们有对`kotlin.text.HexFormat.Companion`的引用，我们在UAST中解析它，我们得到一个`PsiMethod`。这**不是**Kotlin PSI元素，所以我们之前检查这是否是伴生对象的代码（`source is KtObjectDeclaration && source.isCompanion()`）不工作——第一个实例检查失败。这些编译的`PsiElement`不给我们访问我们通常可以在`KtElement`上检查的任何特殊Kotlin有效负载——修饰符如`inline`或`infix`、默认参数等。

分析API正确处理这个，即使对于编译代码。事实上，检查`Nothing`类型的早期实现证明了这一点，因为它分析的方法来自Kotlin标准库（`error`、`TODO`等），这些都是Kotlin标准库jar文件中的编译类！

因此，是的，我们可以使用Kotlin PSI来检查类是否是伴生对象，如果我们实际上有类的源代码。但如果我们解析对类的_引用_，使用Kotlin分析API更好；它对源和编译都有效：

```kotlin
symbol is KaClassSymbol && symbol.classKind == KaClassKind.COMPANION_OBJECT
```

旧API（8.7.0-alpha04之前）：

```kotlin
symbol is KtClassOrObjectSymbol && symbol.classKind == KtClassKind.COMPANION_OBJECT
```

##### Kotlin分析API操作指南

当你在lint中使用K2时，很多UAST对Kotlin中解析和类型的处理实际上在幕后使用分析API。

如果你例如有一个Kotlin PSI `KtThisExpression`，你想要理解如何将`this`引用解析到另一个PSI元素，写以下Kotlin UAST代码：

```kotlin
thisReference.toUElement()?.tryResolve()
```

你现在可以使用调试器步入`tryResolve`调用，你最终会在使用Kotlin分析API查找它的代码中，事实证明，这是如何：

```kotlin
analyze(expression) {
    val reference = expression.getTargetLabel()?.mainReference
        ?: expression.instanceReference.mainReference
    val psi = reference.resolveToSymbol()?.psi
    …
}
```

#### 配置lint使用K2

要从单元测试使用K2，你可以使用以下lint测试任务覆盖：

```kotlin
override fun lint(): TestLintTask {
    return super.lint().configureOptions { flags -> flags.setUseK2Uast(true) }
}
```

在测试外，你也可以在运行配置中设置`-Dlint.use.fir.uast=true`系统属性。

注意在某个时候这个标志可能会消失，因为我们会完全切换到K2。

### API兼容性

8.7.0-alpha04之前的lint版本使用分析API的旧版本。如果你的lint检查正在针对这些旧版本构建，你需要使用旧名称和API，如`KtSymbol`和`KtType`而不是`KaSymbol`和`KaType`。

分析API不稳定，在这些版本之间显著变化。希望/计划是API很快稳定，这样你可以开始在lint检查中使用分析API，并让它与lint的未来版本一起工作。

现在，lint使用特殊的字节码重写器即时尝试自动迁移使用旧API的编译lint检查，但这不处理所有边缘情况，所以最好的前进路径是使用新API。在下面的食谱中，我们暂时显示新版本和旧版本。

### 食谱

这里是各种其他Kotlin分析场景和潜在解决方案：

#### 解析函数调用

```kotlin
val call: KtCallExpression
…
analyze(call) {
    val callInfo = call.resolveToCall() ?: return null
    val symbol: KaFunctionSymbol? = callInfo.singleFunctionCallOrNull()?.symbol
        ?: callInfo.singleConstructorCallOrNull()?.symbol
        ?: callInfo.singleCallOrNull<KaAnnotationCall>()?.symbol
    …
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
val call: KtCallExpression
…
analyze(call) {
    val callInfo = call.resolveCall()
    if (callInfo != null) {
      val symbol: KtFunctionLikeSymbol = callInfo.singleFunctionCallOrNull()?.symbol
          ?: callInfo.singleConstructorCallOrNull()?.symbol
          ?: callInfo.singleCallOrNull<KtAnnotationCall>()?.symbol
      …
}
```

#### 解析变量引用

也使用`resolveCall`，尽管它实际上不是调用：

```kotlin
val expression: KtNameReferenceExpression
…
analyze(expression) {
    val symbol: KaVariableSymbol? = expression.resolveToCall()?.singleVariableAccessCall()?.symbol
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
val expression: KtNameReferenceExpression
…
analyze(expression) {
    val symbol: KtVariableLikeSymbol = expression.resolveCall()?.singleVariableAccessCall()?.symbol
}
```

#### 获取符号的包含类

```kotlin
val containingSymbol = symbol.containingSymbol
if (containingSymbol is KaNamedClassSymbol) {
    …
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
val containingSymbol = symbol.getContainingSymbol()
if (containingSymbol is KtNamedClassOrObjectSymbol) {
    …
}
```

#### 获取类的完全限定名

```kotlin
val containing = declarationSymbol.containingSymbol
if (containing is KaClassSymbol) {
    val fqn = containing.classId?.asSingleFqName()
    …
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
val containing = declarationSymbol.getContainingSymbol()
if (containing is KtClassOrObjectSymbol) {
    val fqn = containing.classIdIfNonLocal?.asSingleFqName()
    …
}
```

#### 查找符号的弃用状态

```kotlin
if (symbol is KaDeclarationSymbol) {
    symbol.deprecationStatus?.let { … }
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
if (symbol is KtDeclarationSymbol) {
    symbol.deprecationStatus?.let { … }
}
```

#### 查找可见性

```kotlin
if (symbol is KaDeclarationSymbol) {
    if (!isPublicApi(symbol)) {
        …
    }
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
if (symbol is KtSymbolWithVisibility) {
    val visibility = symbol.visibility
    if (!visibility.isPublicAPI) {
        …
    }
}
```

#### 获取类符号的KtType

```kotlin
if (symbol is KaNamedClassSymbol) {
    val type = symbol.defaultType
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
containingSymbol.buildSelfClassType()
```

#### 将KtType解析为类

示例：这个`KtParameter`指向接口吗？

```kotlin
analyze(ktParameter) {
    val parameterSymbol = ktParameter.symbol
    val returnType = parameterSymbol.returnType
    val typeSymbol = returnType.expandedSymbol
    if (typeSymbol is KaClassSymbol) {
        val classKind = typeSymbol.classKind
        if (classKind == KaClassKind.INTERFACE) {
            …
        }
    }
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
analyze(ktParameter) {
    val parameterSymbol = ktParameter.getParameterSymbol()
    val returnType = parameterSymbol.returnType
    val typeSymbol = returnType.expandedClassSymbol
    if (typeSymbol is KtClassOrObjectSymbol) {
    val classKind = typeSymbol.classKind
    if (classKind == KtClassKind.INTERFACE) {
        …
    }
}
```

#### 看两个类型是否引用相同的原始类（擦除）：

```kotlin
if (type1 is KaClassType && type2 is KaClassType &&
    type1.classId == type2.classId
) {
    …
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
if (type1 is KtNonErrorClassType && type2 is KtNonErrorClassType &&
    type1.classId == type2.classId) {
    …
}
```

#### 对于扩展方法，获取接收器类型：

```kotlin
if (declarationSymbol is KaNamedFunctionSymbol) {
    val declarationReceiverType = declarationSymbol.receiverParameter?.type
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
if (declarationSymbol is KtFunctionSymbol) {
    val declarationReceiverType = declarationSymbol.receiverParameter?.type
```

#### 获取包含符号声明的PsiFile

```kotlin
val file = symbol.containingFile
if (file != null) {
    val psi = file.psi
    if (psi is PsiFile) {
        …
    }
}
```

旧API（8.7.0-alpha04之前）：

```kotlin
val file = symbol.getContainingFileSymbol()
if (file is KtFileSymbol) {
val psi = file.psi
if (psi is PsiFile) {
     …
}
```
