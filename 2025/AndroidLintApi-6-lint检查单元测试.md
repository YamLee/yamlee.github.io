# Android Lint API 指南 - 第6章：Lint检查单元测试


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第6章节"Lint Check Unit Testing"的中文翻译版本。

## Lint检查单元测试

为你的lint检查编写单元测试至关重要。Lint提供了一个专门的测试框架，使编写和维护测试变得简单。

### 基本测试设置

#### 依赖配置

首先，在你的`build.gradle`文件中添加测试依赖：

```groovy
dependencies {
    compileOnly "com.android.tools.lint:lint-api:$lintVersion"
    testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
    testImplementation 'junit:junit:4.13.2'
}
```

#### 基本测试结构

一个典型的lint检查测试如下所示：

```kotlin
import com.android.tools.lint.checks.infrastructure.TestFiles.java
import com.android.tools.lint.checks.infrastructure.TestFiles.kotlin
import com.android.tools.lint.checks.infrastructure.TestLintTask.lint
import org.junit.Test

class MyDetectorTest {
    @Test
    fun testBasicDetection() {
        lint()
            .files(
                kotlin("""
                    package test.pkg
                    
                    class TestClass {
                        fun problematicMethod() {
                            // 这里是触发lint检查的代码
                        }
                    }
                """).indented()
            )
            .issues(MyDetector.ISSUE)
            .run()
            .expect("""
                src/test/pkg/TestClass.kt:5: Warning: 问题描述 [IssueId]
                        // 这里是触发lint检查的代码
                        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                0 errors, 1 warnings
            """.trimIndent())
    }
}
```

### 测试文件创建

#### Kotlin文件

```kotlin
kotlin("""
    package test.pkg
    
    class KotlinClass {
        val property = "value"
        
        fun method() {
            println("Hello")
        }
    }
""").indented()
```

#### Java文件

```kotlin
java("""
    package test.pkg;
    
    public class JavaClass {
        private String field = "value";
        
        public void method() {
            System.out.println("Hello");
        }
    }
""").indented()
```

#### XML文件

```kotlin
xml("res/layout/activity_main.xml", """
    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Hello World!" />
            
    </LinearLayout>
""").indented()
```

#### Manifest文件

```kotlin
manifest("""
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="test.pkg">
        
        <application android:label="Test App">
            <activity android:name=".MainActivity">
                <intent-filter>
                    <action android:name="android.intent.action.MAIN" />
                    <category android:name="android.intent.category.LAUNCHER" />
                </intent-filter>
            </activity>
        </application>
    </manifest>
""").indented()
```

#### Gradle文件

```kotlin
gradle("""
    apply plugin: 'com.android.application'
    
    android {
        compileSdkVersion 30
        defaultConfig {
            applicationId "test.pkg"
            minSdkVersion 21
            targetSdkVersion 30
        }
    }
    
    dependencies {
        implementation 'androidx.appcompat:appcompat:1.3.0'
    }
""").indented()
```

### 测试配置

#### 指定问题

```kotlin
lint()
    .files(/* 测试文件 */)
    .issues(MyDetector.ISSUE1, MyDetector.ISSUE2)  // 测试多个问题
    .run()
```

#### 设置项目类型

```kotlin
lint()
    .files(/* 测试文件 */)
    .issues(MyDetector.ISSUE)
    .sdkHome(TestUtils.getSdk())  // 设置SDK路径
    .run()
```

#### 配置选项

```kotlin
lint()
    .files(/* 测试文件 */)
    .issues(MyDetector.ISSUE)
    .configureOptions { options ->
        options.enable(MyDetector.ISSUE)
        options.setSeverityOverride(MyDetector.ISSUE, Severity.ERROR)
    }
    .run()
```

### 期望结果验证

#### 期望特定输出

```kotlin
.expect("""
    src/test/pkg/TestClass.kt:5: Warning: 具体的警告消息 [IssueId]
            problematicCode()
            ~~~~~~~~~~~~~~~~~
    0 errors, 1 warnings
""".trimIndent())
```

#### 期望无问题

```kotlin
.expectClean()
```

#### 期望错误计数

```kotlin
.expectErrorCount(2)
.expectWarningCount(1)
```

#### 期望修复后的结果

```kotlin
.expectFixDiffs("""
    Fix for src/test/pkg/TestClass.kt line 5: 修复建议:
    @@ -3 +3
    -     problematicCode()
    +     fixedCode()
""".trimIndent())
```

### 高级测试场景

#### 测试多个文件

```kotlin
@Test
fun testMultipleFiles() {
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
                        ClassA().methodA()  // 可能的问题
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect(/* 期望结果 */)
}
```

#### 测试抑制

```kotlin
@Test
fun testSuppression() {
    lint()
        .files(
            kotlin("""
                package test.pkg
                
                class TestClass {
                    @Suppress("MyIssueId")
                    fun suppressedMethod() {
                        // 这个问题应该被抑制
                    }
                    
                    fun normalMethod() {
                        // 这个问题不应该被抑制
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect("""
            src/test/pkg/TestClass.kt:10: Warning: 问题描述 [MyIssueId]
                    // 这个问题不应该被抑制
                    ~~~~~~~~~~~~~~~~~~~~~~~~
            0 errors, 1 warnings
        """.trimIndent())
}
```

#### 测试配置文件

```kotlin
@Test
fun testWithLintXml() {
    lint()
        .files(
            kotlin(/* 测试代码 */),
            lintXml("""
                <?xml version="1.0" encoding="UTF-8"?>
                <lint>
                    <issue id="MyIssueId" severity="error" />
                </lint>
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect(/* 期望错误而不是警告 */)
}
```

### 测试存根和模拟

#### 创建API存根

当你的检查依赖于特定的Android API或第三方库时，需要创建存根：

```kotlin
@Test
fun testWithStubs() {
    lint()
        .files(
            // 创建Android API存根
            java("""
                package android.content;
                public class Context {
                    public String getPackageName() { return null; }
                }
            """).indented(),
            
            // 创建第三方库存根
            java("""
                package com.example.library;
                public class LibraryClass {
                    public void libraryMethod() { }
                }
            """).indented(),
            
            // 实际测试代码
            kotlin("""
                package test.pkg
                import android.content.Context
                import com.example.library.LibraryClass
                
                class TestClass(private val context: Context) {
                    fun useLibrary() {
                        LibraryClass().libraryMethod()
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect(/* 期望结果 */)
}
```

#### 使用预定义存根

Lint测试框架提供了一些常用的存根：

```kotlin
import com.android.tools.lint.checks.infrastructure.TestFiles.*

@Test
fun testWithBuiltInStubs() {
    lint()
        .files(
            // 使用内置的支持库存根
            SUPPORT_ANNOTATIONS_JAR,
            
            // 你的测试代码
            kotlin("""
                package test.pkg
                import androidx.annotation.NonNull
                
                class TestClass {
                    fun method(@NonNull param: String) {
                        // 测试代码
                    }
                }
            """).indented()
        )
        .issues(MyDetector.ISSUE)
        .run()
        .expect(/* 期望结果 */)
}
```

### 测试最佳实践

#### 1. 测试边界情况

```kotlin
@Test
fun testEdgeCases() {
    // 测试空文件
    lint()
        .files(kotlin("").indented())
        .issues(MyDetector.ISSUE)
        .run()
        .expectClean()
    
    // 测试只有注释的文件
    lint()
        .files(kotlin("// 只有注释").indented())
        .issues(MyDetector.ISSUE)
        .run()
        .expectClean()
}
```

#### 2. 测试不同语言

```kotlin
@Test
fun testJavaAndKotlin() {
    // 确保检查在Java和Kotlin中都工作
    val javaResult = lint()
        .files(java("""
            public class JavaTest {
                public void problematicMethod() { }
            }
        """).indented())
        .issues(MyDetector.ISSUE)
        .run()
    
    val kotlinResult = lint()
        .files(kotlin("""
            class KotlinTest {
                fun problematicMethod() { }
            }
        """).indented())
        .issues(MyDetector.ISSUE)
        .run()
    
    // 两者应该产生相似的结果
}
```

#### 3. 测试性能

```kotlin
@Test
fun testPerformance() {
    val startTime = System.currentTimeMillis()
    
    lint()
        .files(
            // 创建大量测试文件
            *(1..100).map { i ->
                kotlin("""
                    package test.pkg$i
                    class TestClass$i {
                        fun method$i() { }
                    }
                """).indented()
            }.toTypedArray()
        )
        .issues(MyDetector.ISSUE)
        .run()
    
    val endTime = System.currentTimeMillis()
    val duration = endTime - startTime
    
    // 确保检查在合理时间内完成
    assertTrue("检查耗时过长: ${duration}ms", duration < 5000)
}
```

#### 4. 参数化测试

```kotlin
@RunWith(Parameterized::class)
class MyDetectorParameterizedTest(
    private val testCase: String,
    private val expectedWarnings: Int
) {
    companion object {
        @JvmStatic
        @Parameterized.Parameters(name = "{0}")
        fun data() = listOf(
            arrayOf("正常情况", 0),
            arrayOf("单个问题", 1),
            arrayOf("多个问题", 3)
        )
    }
    
    @Test
    fun testParameterized() {
        lint()
            .files(getTestFile(testCase))
            .issues(MyDetector.ISSUE)
            .run()
            .expectWarningCount(expectedWarnings)
    }
    
    private fun getTestFile(testCase: String): TestFile {
        // 根据测试案例返回相应的测试文件
        return when (testCase) {
            "正常情况" -> kotlin("class Normal { }").indented()
            "单个问题" -> kotlin("class SingleIssue { fun bad() { } }").indented()
            "多个问题" -> kotlin("""
                class MultipleIssues { 
                    fun bad1() { }
                    fun bad2() { }
                    fun bad3() { }
                }
            """).indented()
            else -> kotlin("").indented()
        }
    }
}
```

### 调试测试

#### 打印实际输出

```kotlin
@Test
fun debugTest() {
    val result = lint()
        .files(/* 测试文件 */)
        .issues(MyDetector.ISSUE)
        .run()
    
    // 打印实际输出以便调试
    println("实际输出:")
    println(result.output)
    
    // 然后设置期望
    result.expect(/* 期望结果 */)
}
```

#### 使用断点

在检测器代码中设置断点，然后以调试模式运行测试，可以逐步检查检测器的执行过程。

### 持续集成

#### GitHub Actions配置

```yaml
name: Lint Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up JDK 11
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
    
    - name: Run Lint Tests
      run: ./gradlew :checks:test
    
    - name: Upload Test Results
      uses: actions/upload-artifact@v2
      if: always()
      with:
        name: test-results
        path: checks/build/reports/tests/
```
