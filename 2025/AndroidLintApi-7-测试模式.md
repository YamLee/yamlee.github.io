# Android Lint API 指南 - 第7章：测试模式


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第7章节"Test Modes"的中文翻译版本。

## 测试模式

Lint测试框架提供了多种测试模式，允许你以不同的方式运行和验证你的检查器。了解这些模式可以帮助你编写更全面和有效的测试。

### 基本运行模式

#### 标准运行模式

这是最常见的测试模式，运行完整的lint分析：

```kotlin
@Test
fun testStandardMode() {
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
        .run()  // 标准运行模式
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

#### 增量模式

增量模式模拟IDE中的实时分析，只分析当前文件：

```kotlin
@Test
fun testIncrementalMode() {
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
        .incremental()  // 增量模式
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

### 作用域测试

#### 单文件作用域

测试检查器在只有当前文件可用时的行为：

```kotlin
@Test
fun testSingleFileScope() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        // 这个检查应该只看当前文件
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .incremental("src/test/pkg/TestClass.kt")  // 指定单个文件
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

#### 多文件作用域

测试需要访问多个文件的检查器：

```kotlin
@Test
fun testMultiFileScope() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class ClassA {
                    fun methodA() { }
                }
            """).indented(),
            kotlin("""
                package test.pkg
                class ClassB {
                    fun methodB() {
                        ClassA().methodA()  // 跨文件引用
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()  // 完整分析，包含所有文件
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

### 配置测试模式

#### 启用/禁用检查

```kotlin
@Test
fun testEnabledDisabled() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .configureOptions { options ->
            options.disable(MyDetector.ISSUE)  // 禁用检查
        }
        .run()
        .expectClean()  // 应该没有警告
        
    // 测试启用
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .configureOptions { options ->
            options.enable(MyDetector.ISSUE)  // 启用检查
        }
        .run()
        .expect("""
            // 期望的警告
        """.trimIndent())
}
```

#### 严重性覆盖

```kotlin
@Test
fun testSeverityOverride() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun problematicMethod() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .configureOptions { options ->
            // 将警告提升为错误
            options.setSeverityOverride(MyDetector.ISSUE, Severity.ERROR)
        }
        .run()
        .expect("""
            src/test/pkg/TestClass.kt:3: Error: 问题描述 [IssueId]
                fun problematicMethod() { }
                ~~~~~~~~~~~~~~~~~~~~~~~~~~~
            1 errors, 0 warnings
        """.trimIndent())
}
```

### 部分分析模式

#### 测试部分分析

从Lint 7.0开始，支持部分分析模式，这对CI环境特别有用：

```kotlin
@Test
fun testPartialAnalysis() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
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

#### 模拟CI环境

```kotlin
@Test
fun testCiEnvironment() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .configureOptions { options ->
            // 模拟CI环境的配置
            options.setCheckGeneratedSources(false)
            options.setCheckTestSources(false)
        }
        .partial()
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

### 平台和SDK测试

#### 指定目标SDK

```kotlin
@Test
fun testWithSpecificSdk() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                import android.os.Build
                
                class TestClass {
                    fun method() {
                        if (Build.VERSION.SDK_INT >= 21) {
                            // API 21+ 特定的代码
                        }
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .sdkHome(TestUtils.getSdk())
        .compileSdkVersion(30)
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

#### 测试不同的构建类型

```kotlin
@Test
fun testDebugVsRelease() {
    // 测试Debug构建
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun debugMethod() {
                        // Debug特定的代码
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .variant("debug")
        .run()
        .expect("""
            // Debug构建的期望输出
        """.trimIndent())
        
    // 测试Release构建
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun releaseMethod() {
                        // Release特定的代码
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .variant("release")
        .run()
        .expect("""
            // Release构建的期望输出
        """.trimIndent())
}
```

### 网络和资源测试

#### 模拟网络请求

```kotlin
@Test
fun testWithNetworkData() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun checkVersion() {
                        // 代码中可能包含版本检查
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .networkData("https://maven.google.com/master-index.xml", """
            <?xml version='1.0' encoding='UTF-8'?>
            <metadata>
                <com.android.tools.build>
                    <gradle versions="7.0.0,7.1.0"/>
                </com.android.tools.build>
            </metadata>
        """.trimIndent())
        .run()
        .expect("""
            // 基于网络数据的期望输出
        """.trimIndent())
}
```

#### 测试资源文件

```kotlin
@Test
fun testWithResources() {
    lint()
        .files(
            xml("res/values/strings.xml", """
                <?xml version="1.0" encoding="utf-8"?>
                <resources>
                    <string name="app_name">Test App</string>
                    <string name="welcome_message">Welcome!</string>
                </resources>
            """).indented(),
            kotlin("""
                package test.pkg
                class TestClass {
                    fun getString() {
                        // 使用字符串资源的代码
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect("""
            // 期望的输出
        """.trimIndent())
}
```

### 性能测试模式

#### 基准测试

```kotlin
@Test
fun testPerformance() {
    val testFiles = (1..100).map { i ->
        kotlin("""
            package test.pkg$i
            class TestClass$i {
                fun method$i() {
                    // 测试方法
                }
            }
        """).indented()
    }.toTypedArray()
    
    val startTime = System.currentTimeMillis()
    
    lint()
        .files(*testFiles)
        .issues(MyDetector.ISSUE)
        .run()
    
    val duration = System.currentTimeMillis() - startTime
    
    // 验证性能在可接受范围内
    assertTrue("分析耗时过长: ${duration}ms", duration < 10000)
}
```

#### 内存使用测试

```kotlin
@Test
fun testMemoryUsage() {
    val runtime = Runtime.getRuntime()
    val initialMemory = runtime.totalMemory() - runtime.freeMemory()
    
    lint()
        .files(
            // 大量测试文件
            *(1..1000).map { i ->
                kotlin("""
                    package test.pkg$i
                    class LargeClass$i {
                        ${(1..100).joinToString("\n") { j -> 
                            "fun method${i}_$j() { /* method body */ }"
                        }}
                    }
                """).indented()
            }.toTypedArray()
        )
        .issues(MyDetector.ISSUE)
        .run()
    
    System.gc()  // 建议进行垃圾回收
    Thread.sleep(100)  // 等待GC完成
    
    val finalMemory = runtime.totalMemory() - runtime.freeMemory()
    val memoryIncrease = finalMemory - initialMemory
    
    // 验证内存使用在合理范围内
    assertTrue("内存使用过多: ${memoryIncrease / 1024 / 1024}MB", 
               memoryIncrease < 500 * 1024 * 1024)  // 500MB限制
}
```

### 错误处理测试

#### 测试异常处理

```kotlin
@Test
fun testExceptionHandling() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        // 可能导致检测器异常的代码
                        val problematicCode = null
                        problematicCode!!.toString()
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        // 确保检测器不会因为异常而崩溃
        .expectClean()  // 或者期望适当的错误处理
}
```

#### 测试损坏的源文件

```kotlin
@Test
fun testCorruptedSource() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() {
                        // 语法错误的代码
                        val x = 
                    // 缺少值和闭合括号
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .allowCompilationErrors()  // 允许编译错误
        .run()
        .expectClean()  // 检测器应该优雅地处理错误
}
```

### 自定义测试模式

#### 创建自定义测试配置

```kotlin
class CustomLintTest {
    
    private fun createCustomLintTask(): TestLintTask {
        return lint()
            .configureOptions { options ->
                // 自定义配置
                options.setCheckGeneratedSources(true)
                options.setCheckTestSources(true)
                options.setCheckDependencies(false)
            }
            .sdkHome(TestUtils.getSdk())
            .compileSdkVersion(31)
    }
    
    @Test
    fun testWithCustomConfiguration() {
        createCustomLintTask()
            .files(
                kotlin("""
                    package test.pkg
                    class TestClass {
                        fun method() { }
                    }
                """).indented()
            )
            .issues(MyDetector.ISSUE)
            .run()
            .expect("""
                // 期望的输出
            """.trimIndent())
    }
}
```

#### 参数化测试模式

```kotlin
@RunWith(Parameterized::class)
class ParameterizedModeTest(
    private val testMode: String,
    private val configureLint: (TestLintTask) -> TestLintTask
) {
    
    companion object {
        @JvmStatic
        @Parameterized.Parameters(name = "{0}")
        fun testModes() = listOf(
            arrayOf("标准模式") { task: TestLintTask -> task },
            arrayOf("增量模式") { task: TestLintTask -> task.incremental() },
            arrayOf("部分分析") { task: TestLintTask -> task.partial() },
            arrayOf("严格模式") { task: TestLintTask -> 
                task.configureOptions { it.setSeverityOverride(MyDetector.ISSUE, Severity.ERROR) }
            }
        )
    }
    
    @Test
    fun testInDifferentModes() {
        val result = configureLint(
            lint()
                .files(
                    kotlin("""
                        package test.pkg
                        class TestClass {
                            fun method() { }
                        }
                    """).indented()
                )
                .issues(MyDetector.ISSUE)
        ).run()
        
        // 根据模式验证结果
        when (testMode) {
            "严格模式" -> result.expectErrorCount(1)
            else -> result.expectWarningCount(1)
        }
    }
}
```

### 调试和分析模式

#### 详细输出模式

```kotlin
@Test
fun testWithVerboseOutput() {
    val result = lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .verbose(true)  // 启用详细输出
        .run()
    
    // 打印详细信息用于调试
    println("详细输出:")
    println(result.output)
    
    result.expect("""
        // 期望的输出
    """.trimIndent())
}
```

#### 分析统计

```kotlin
@Test
fun testWithStatistics() {
    val result = lint()
        .files(
            kotlin("""
                package test.pkg
                class TestClass {
                    fun method() { }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
    
    // 获取分析统计信息
    val stats = result.stats
    println("文件数量: ${stats.fileCount}")
    println("分析时间: ${stats.analysisTimeMs}ms")
    println("问题数量: ${stats.issueCount}")
}
```
