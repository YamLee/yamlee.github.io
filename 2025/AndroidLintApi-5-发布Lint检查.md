# Android Lint API 指南 - 第5章：发布Lint检查


本文档是 [Android Custom Lint Rules API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.md.html) 第5章节"Publishing a Lint Check"的中文翻译版本。

## 发布Lint检查

一旦你编写了lint检查，你需要让它可供其他开发者使用。

有几种方式可以做到这一点：

1. **通过AAR文件**：将lint检查打包到Android库中
2. **作为独立JAR**：创建可以直接使用的独立lint规则JAR
3. **通过Maven/Gradle依赖**：发布到Maven Central或其他仓库

### 通过AAR文件发布

这是最常见的方法，特别是当你的lint检查与特定的Android库相关时。

#### 设置项目结构

创建一个包含以下模块的项目：

```
project/
├── checks/          # lint检查实现
├── library/         # Android库
└── sample/          # 示例应用（可选）
```

#### checks模块

这是实际的lint检查实现模块：

```groovy
// checks/build.gradle
apply plugin: 'java-library'
apply plugin: 'kotlin'

dependencies {
    compileOnly "com.android.tools.lint:lint-api:$lintVersion"
    compileOnly "com.android.tools.lint:lint-checks:$lintVersion"
    
    testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
    testImplementation 'junit:junit:4.13.2'
}

// 确保Kotlin编译兼容性
compileKotlin {
    kotlinOptions.jvmTarget = "1.8"
}
compileTestKotlin {
    kotlinOptions.jvmTarget = "1.8"
}

jar {
    manifest {
        attributes("Lint-Registry-v2": "com.example.lint.MyIssueRegistry")
    }
}
```

#### library模块

这是Android库模块，它将包含并发布lint检查：

```groovy
// library/build.gradle
apply plugin: 'com.android.library'
apply plugin: 'kotlin-android'

android {
    compileSdkVersion 34
    
    defaultConfig {
        minSdkVersion 21
        targetSdkVersion 34
        versionCode 1
        versionName "1.0"
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {
    // 将lint检查作为lintPublish依赖添加
    lintPublish project(':checks')
    
    // 其他库依赖...
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
}
```

关键点是`lintPublish project(':checks')`依赖。这告诉Gradle将lint检查包含在生成的AAR文件中。

#### 发布配置

如果你想发布到Maven仓库，添加发布配置：

```groovy
// library/build.gradle
apply plugin: 'maven-publish'

afterEvaluate {
    publishing {
        publications {
            release(MavenPublication) {
                from components.release
                
                groupId = 'com.example'
                artifactId = 'my-library'
                version = '1.0.0'
                
                pom {
                    name = 'My Library'
                    description = 'A library with custom lint checks'
                    url = 'https://github.com/example/my-library'
                    
                    licenses {
                        license {
                            name = 'The Apache License, Version 2.0'
                            url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                        }
                    }
                    
                    developers {
                        developer {
                            id = 'developer-id'
                            name = 'Developer Name'
                            email = 'developer@example.com'
                        }
                    }
                    
                    scm {
                        connection = 'scm:git:git://github.com/example/my-library.git'
                        developerConnection = 'scm:git:ssh://github.com/example/my-library.git'
                        url = 'https://github.com/example/my-library'
                    }
                }
            }
        }
        
        repositories {
            maven {
                name = "sonatype"
                url = "https://oss.sonatype.org/service/local/staging/deploy/maven2/"
                credentials {
                    username = ossrhUsername
                    password = ossrhPassword
                }
            }
        }
    }
}
```

### 作为独立JAR发布

如果你的lint检查不依赖于特定的Android库，你可以将其作为独立JAR发布：

```groovy
// checks/build.gradle
apply plugin: 'java-library'
apply plugin: 'kotlin'
apply plugin: 'maven-publish'

dependencies {
    compileOnly "com.android.tools.lint:lint-api:$lintVersion"
    testImplementation "com.android.tools.lint:lint-tests:$lintVersion"
}

jar {
    manifest {
        attributes("Lint-Registry-v2": "com.example.lint.MyIssueRegistry")
    }
}

publishing {
    publications {
        maven(MavenPublication) {
            from components.java
            
            groupId = 'com.example'
            artifactId = 'custom-lint-checks'
            version = '1.0.0'
        }
    }
}
```

### 使用发布的Lint检查

#### 从AAR使用

当其他项目依赖你的库时，lint检查会自动可用：

```groovy
dependencies {
    implementation 'com.example:my-library:1.0.0'
}
```

#### 从独立JAR使用

对于独立的lint检查JAR：

```groovy
dependencies {
    lintChecks 'com.example:custom-lint-checks:1.0.0'
}
```

### 本地测试

在发布之前，你可以在本地测试你的lint检查：

#### 使用本地AAR

```groovy
// 在消费项目中
dependencies {
    implementation project(':library')  // 本地库项目
}
```

#### 使用本地JAR

```groovy
dependencies {
    lintChecks files('path/to/your/lint-checks.jar')
}
```

### Lint检查版本兼容性

确保你的lint检查与目标Android Gradle Plugin版本兼容：

```kotlin
// 在IssueRegistry中
class MyIssueRegistry : IssueRegistry() {
    override val api: Int = CURRENT_API
    override val minApi: Int = 8  // 支持的最低API版本
    
    override val vendor: Vendor = Vendor(
        vendorName = "Example Company",
        feedbackUrl = "https://github.com/example/issues",
        contact = "support@example.com"
    )
    
    override val issues = listOf(
        MyDetector.ISSUE
    )
}
```

### 最佳实践

#### 1. 版本管理

使用语义化版本控制：
- 主版本号：不兼容的API更改
- 次版本号：向后兼容的功能添加
- 修订版本号：向后兼容的错误修复

#### 2. 文档

提供清晰的文档：
- README文件解释如何使用
- 每个检查的详细说明
- 配置选项和抑制方法

#### 3. 测试覆盖

确保全面的测试覆盖：
```kotlin
class MyDetectorTest {
    @Test
    fun testDetection() {
        lint()
            .files(
                kotlin("""
                    // 测试代码
                """).indented()
            )
            .issues(MyDetector.ISSUE)
            .run()
            .expect("""
                // 期望的输出
            """.trimIndent())
    }
    
    @Test
    fun testSuppression() {
        lint()
            .files(
                kotlin("""
                    @Suppress("MyIssueId")
                    // 应该被抑制的代码
                """).indented()
            )
            .issues(MyDetector.ISSUE)
            .run()
            .expectClean()
    }
}
```

#### 4. 性能考虑

- 避免昂贵的操作
- 使用适当的作用域
- 缓存重复计算的结果

#### 5. 错误处理

```kotlin
class MyDetector : Detector(), SourceCodeScanner {
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        try {
            // 检查逻辑
        } catch (e: Exception) {
            // 记录错误但不要崩溃
            context.log(e, "Error in MyDetector")
        }
    }
}
```

### CI/CD集成

#### GitHub Actions示例

```yaml
name: Build and Test

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
    
    - name: Cache Gradle packages
      uses: actions/cache@v2
      with:
        path: ~/.gradle/caches
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}
        restore-keys: ${{ runner.os }}-gradle
    
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
    
    - name: Run tests
      run: ./gradlew test
    
    - name: Build
      run: ./gradlew build
    
    - name: Publish to Maven Central
      if: github.ref == 'refs/heads/main'
      env:
        OSSRH_USERNAME: ${{ secrets.OSSRH_USERNAME }}
        OSSRH_PASSWORD: ${{ secrets.OSSRH_PASSWORD }}
        SIGNING_KEY_ID: ${{ secrets.SIGNING_KEY_ID }}
        SIGNING_PASSWORD: ${{ secrets.SIGNING_PASSWORD }}
        SIGNING_SECRET_KEY_RING_FILE: ${{ secrets.SIGNING_SECRET_KEY_RING_FILE }}
      run: ./gradlew publishToSonatype
```

### 故障排除

#### 常见问题

1. **Lint检查未被识别**
   - 检查服务注册文件是否正确
   - 确认JAR中包含META-INF/services文件

2. **版本兼容性问题**
   - 使用正确的lint API版本
   - 设置适当的minApi和api值

3. **在IDE中不工作**
   - 确保作用域设置正确
   - 检查是否需要多个作用域

4. **性能问题**
   - 分析检查的复杂度
   - 考虑使用更具体的回调而不是通用访问者
