# Android Lint API 指南 - 注解

## 注解

注解是Android Lint中一个重要的概念，它们提供了关于代码意图和约束的元数据信息。Lint可以利用这些注解来提供更准确的分析和建议。

### 支持库注解

Android支持库提供了一系列有用的注解，Lint可以理解并利用这些注解：

#### @Nullable 和 @NonNull

这些注解用于指示参数、返回值或字段是否可以为null：

```java
public class UserManager {
    
    @Nullable
    public String getUserName(@NonNull String userId) {
        if (userId.isEmpty()) { // Lint知道userId不会为null
            return null;
        }
        return database.findUser(userId);
    }
    
    public void updateUser(@NonNull User user) {
        // Lint会检查传入的user是否可能为null
        database.save(user);
    }
}
```

#### @IntRange 和 @FloatRange

用于指定数值参数的有效范围：

```java
public class VolumeController {
    
    public void setVolume(@IntRange(from = 0, to = 100) int volume) {
        // volume必须在0到100之间
        this.volume = volume;
    }
    
    public void setAlpha(@FloatRange(from = 0.0, to = 1.0) float alpha) {
        // alpha必须在0.0到1.0之间
        this.alpha = alpha;
    }
}
```

#### @Size

用于指定集合、数组或字符串的大小约束：

```java
public class LocationUtils {
    
    public void setCoordinates(@Size(2) double[] coords) {
        // coords数组必须恰好有2个元素
        this.latitude = coords[0];
        this.longitude = coords[1];
    }
    
    public void setRgbColor(@Size(min = 3, max = 4) int[] color) {
        // color数组必须有3或4个元素（RGB或RGBA）
        this.color = color;
    }
    
    public void validatePassword(@Size(min = 8) String password) {
        // password至少需要8个字符
        this.password = password;
    }
}
```

#### @RequiresPermission

用于指示方法需要特定的权限：

```java
public class LocationService {
    
    @RequiresPermission(Manifest.permission.ACCESS_FINE_LOCATION)
    public Location getCurrentLocation() {
        // 需要精确位置权限
        return locationManager.getLastKnownLocation(LocationManager.GPS_PROVIDER);
    }
    
    @RequiresPermission(anyOf = {
        Manifest.permission.ACCESS_COARSE_LOCATION,
        Manifest.permission.ACCESS_FINE_LOCATION
    })
    public Location getApproximateLocation() {
        // 需要粗略或精确位置权限之一
        return locationManager.getLastKnownLocation(LocationManager.NETWORK_PROVIDER);
    }
}
```

#### @RequiresApi

用于指示方法需要特定的API级别：

```java
public class NotificationHelper {
    
    @RequiresApi(api = Build.VERSION_CODES.O)
    public void createNotificationChannel() {
        // 只在API 26+上可用
        NotificationChannel channel = new NotificationChannel(
            CHANNEL_ID, 
            "Channel Name", 
            NotificationManager.IMPORTANCE_DEFAULT
        );
        notificationManager.createNotificationChannel(channel);
    }
}
```

### 线程注解

用于指示方法应该在哪个线程上调用：

```java
public class DataRepository {
    
    @MainThread
    public void updateUI() {
        // 必须在主线程上调用
        textView.setText("Updated");
    }
    
    @WorkerThread
    public void performNetworkCall() {
        // 必须在工作线程上调用
        String result = httpClient.get(url);
        processResult(result);
    }
    
    @UiThread
    public void showDialog() {
        // 必须在UI线程上调用
        dialog.show();
    }
    
    @AnyThread
    public String getConfiguration() {
        // 可以在任何线程上调用
        return sharedPreferences.getString("config", "");
    }
}
```

### 资源类型注解

用于指示参数应该是特定类型的资源：

```java
public class ResourceHelper {
    
    public void setBackgroundColor(@ColorRes int colorResId) {
        // colorResId必须是颜色资源ID
        view.setBackgroundColor(ContextCompat.getColor(context, colorResId));
    }
    
    public void loadImage(@DrawableRes int drawableResId) {
        // drawableResId必须是drawable资源ID
        imageView.setImageResource(drawableResId);
    }
    
    public void setText(@StringRes int stringResId) {
        // stringResId必须是字符串资源ID
        textView.setText(context.getString(stringResId));
    }
}
```

### 自定义注解

您也可以创建自定义注解供Lint使用：

```java
@Retention(RetentionPolicy.CLASS)
@Target({ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD})
public @interface Experimental {
    String value() default "";
}

@Retention(RetentionPolicy.CLASS)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface ThreadSafe {
}

@Retention(RetentionPolicy.CLASS)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface NotThreadSafe {
}
```

### 在Lint检查中使用注解

您可以在自定义Lint检查中利用注解信息：

```kotlin
class AnnotationDetector : Detector(), Detector.UastScanner {
    
    companion object {
        val MISSING_PERMISSION = Issue.create(
            id = "MissingPermission",
            briefDescription = "缺少必需的权限",
            explanation = "调用需要权限的方法时必须声明相应权限",
            category = Category.SECURITY,
            priority = 8,
            severity = Severity.ERROR,
            implementation = Implementation(
                AnnotationDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
    
    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        // 检查@RequiresPermission注解
        val annotation = context.evaluator.findAnnotation(
            method, 
            "androidx.annotation.RequiresPermission"
        )
        
        if (annotation != null) {
            val permission = getRequiredPermission(annotation)
            if (permission != null && !hasPermission(context, permission)) {
                context.report(
                    MISSING_PERMISSION,
                    node,
                    context.getLocation(node),
                    "调用此方法需要权限: $permission"
                )
            }
        }
        
        // 检查API级别要求
        checkApiLevel(context, node, method)
        
        // 检查线程要求
        checkThreadRequirement(context, node, method)
    }
    
    private fun getRequiredPermission(annotation: UAnnotation): String? {
        val value = annotation.findAttributeValue("value")
        return if (value is ULiteralExpression) {
            value.value as? String
        } else null
    }
    
    private fun hasPermission(context: JavaContext, permission: String): Boolean {
        // 检查AndroidManifest.xml中是否声明了权限
        val manifest = context.mainProject.manifest
        return manifest?.usesPermissions?.any { 
            it.name == permission 
        } ?: false
    }
    
    private fun checkApiLevel(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        val annotation = context.evaluator.findAnnotation(
            method,
            "androidx.annotation.RequiresApi"
        )
        
        if (annotation != null) {
            val requiredApi = getRequiredApi(annotation)
            val currentApi = context.mainProject.minSdk
            
            if (requiredApi > currentApi) {
                context.report(
                    API_LEVEL_TOO_HIGH,
                    node,
                    context.getLocation(node),
                    "此方法需要API级别$requiredApi，但当前最小API级别是$currentApi"
                )
            }
        }
    }
    
    private fun checkThreadRequirement(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        val threadAnnotations = listOf(
            "androidx.annotation.MainThread",
            "androidx.annotation.UiThread",
            "androidx.annotation.WorkerThread"
        )
        
        for (annotationName in threadAnnotations) {
            val annotation = context.evaluator.findAnnotation(method, annotationName)
            if (annotation != null) {
                val currentThread = getCurrentThreadContext(context, node)
                val requiredThread = getThreadFromAnnotation(annotationName)
                
                if (currentThread != requiredThread && currentThread != ThreadContext.UNKNOWN) {
                    context.report(
                        WRONG_THREAD,
                        node,
                        context.getLocation(node),
                        "此方法必须在${requiredThread}上调用，但当前在${currentThread}上"
                    )
                }
                break
            }
        }
    }
}
```

### 注解处理工具

Lint提供了一些工具方法来处理注解：

```kotlin
class AnnotationUtils {
    
    fun hasAnnotation(
        evaluator: JavaEvaluator,
        element: PsiModifierListOwner,
        annotationName: String
    ): Boolean {
        return evaluator.findAnnotation(element, annotationName) != null
    }
    
    fun getAnnotationValue(
        annotation: UAnnotation,
        attributeName: String = "value"
    ): Any? {
        val attribute = annotation.findAttributeValue(attributeName)
        return when (attribute) {
            is ULiteralExpression -> attribute.value
            is UReferenceExpression -> {
                // 处理常量引用
                val resolved = attribute.resolve()
                if (resolved is PsiField) {
                    evaluateFieldValue(resolved)
                } else null
            }
            else -> null
        }
    }
    
    fun getStringArrayFromAnnotation(
        annotation: UAnnotation,
        attributeName: String
    ): List<String> {
        val attribute = annotation.findAttributeValue(attributeName)
        return when (attribute) {
            is UCallExpression -> {
                // 处理数组字面量
                attribute.valueArguments.mapNotNull { arg ->
                    (arg as? ULiteralExpression)?.value as? String
                }
            }
            is ULiteralExpression -> {
                // 单个值
                listOfNotNull(attribute.value as? String)
            }
            else -> emptyList()
        }
    }
}
```

### 元注解

您可以创建元注解来组合多个约束：

```java
@Retention(RetentionPolicy.CLASS)
@Target({ElementType.METHOD})
@RequiresPermission(Manifest.permission.CAMERA)
@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
public @interface RequiresCameraApi21 {
}

// 使用元注解
public class CameraHelper {
    
    @RequiresCameraApi21
    public void startCameraPreview() {
        // 自动需要相机权限和API 21+
        camera2Manager.startPreview();
    }
}
```

### 注解的最佳实践

1. **一致性**：在整个项目中一致地使用注解
2. **文档化**：注解可以作为API文档的一部分
3. **验证**：使用Lint检查来验证注解约束
4. **性能**：注解处理在编译时进行，不影响运行时性能
5. **版本兼容性**：确保注解库版本与项目兼容

注解是Android开发中强大的工具，它们不仅可以提供编译时检查，还可以改善代码的可读性和维护性。通过合理使用注解，您可以让Lint提供更准确的分析和建议。 