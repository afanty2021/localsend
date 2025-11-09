# Android问题排查

<cite>
**本文档中引用的文件**  
- [app/build.gradle](file://app/android/app/build.gradle)
- [build.gradle](file://app/android/build.gradle)
- [android/build.gradle](file://app/rust_builder/android/build.gradle)
- [Cargo.toml](file://app/rust/Cargo.toml)
- [pubspec.yaml](file://app/pubspec.yaml)
- [run_build_tool.sh](file://app/rust_builder/cargokit/run_build_tool.sh)
- [compile_android_apk.sh](file://scripts/compile_android_apk.sh)
- [settings.gradle](file://app/android/settings.gradle)
- [android_environment.dart](file://app/rust_builder/cargokit/build_tool/lib/src/android_environment.dart)
- [builder.dart](file://app/rust_builder/cargokit/build_tool/lib/src/builder.dart)
</cite>

## 目录
1. [简介](#简介)
2. [NDK版本兼容性问题](#ndk版本兼容性问题)
3. [架构缺失问题](#架构缺失问题)
4. [构建缓存问题](#构建缓存问题)
5. [JNI调用异常与符号未找到](#jni调用异常与符号未找到)
6. [运行时崩溃分析](#运行时崩溃分析)
7. [日志分析工具使用](#日志分析工具使用)
8. [结论](#结论)

## 简介
本文档旨在为Android平台上的Rust集成提供全面的问题排查指南。通过分析项目配置和构建脚本，我们将详细探讨在集成Rust代码时可能遇到的常见问题，包括NDK版本不兼容、目标架构缺失、构建缓存问题以及JNI调用异常等。文档将提供具体的诊断方法和解决方案，帮助开发者快速定位并解决这些问题。

## NDK版本兼容性问题

在Android项目中，NDK（Native Development Kit）版本的兼容性对Rust代码的正确编译至关重要。项目中的`app/android/app/build.gradle`文件明确指定了NDK版本：

```gradle
android {
    namespace "org.localsend.localsend_app"
    compileSdkVersion 34
    ndkVersion flutter.ndkVersion
}
```

同时，在`app/rust_builder/android/build.gradle`中，NDK版本从主应用继承：

```gradle
android {
    // 使用在Flutter项目/app/build.gradle中声明的NDK版本
    ndkVersion android.ndkVersion
}
```

为了检查已安装的NDK版本，可以通过以下步骤进行：

1. 打开Android Studio，进入SDK Manager
2. 查看"SDK Tools"选项卡下的NDK版本
3. 或者在命令行中使用`sdkmanager --list_installed`查看已安装的NDK版本

如果遇到NDK版本不兼容的问题，可以采取以下措施：
- 确保Flutter使用的NDK版本与项目要求一致
- 在`local.properties`文件中明确指定NDK路径
- 使用`sdkmanager`更新到兼容的NDK版本

**Section sources**
- [app/build.gradle](file://app/android/app/build.gradle#L10-L15)
- [android/build.gradle](file://app/rust_builder/android/build.gradle#L28-L31)

## 架构缺失问题

架构缺失问题通常表现为生成的so文件不包含所有目标ABI（Application Binary Interface）。在本项目中，通过以下配置确保支持多个架构：

```gradle
ext.abiCodes = ["x86_64": 1, "armeabi-v7a": 2, "arm64-v8a": 3]
import com.android.build.OutputFile
android.applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def abiVersionCode = project.ext.abiCodes.get(output.getFilter(OutputFile.ABI))
        if (abiVersionCode != null) {
            output.versionCodeOverride = variant.versionCode * 10 + abiVersionCode
        }
    }
}
```

要验证生成的so文件是否包含所有目标ABI，可以：

1. 构建APK后，使用`unzip -l app-release.apk`查看文件内容
2. 检查`lib/`目录下是否包含`x86_64`、`armeabi-v7a`和`arm64-v8a`子目录
3. 确认每个架构目录下都有相应的so文件

如果发现某个架构缺失，可能的原因包括：
- Rust工具链未正确安装对应的目标
- 构建环境缺少必要的交叉编译工具
- NDK版本不支持特定架构

解决方案是确保Rust工具链已安装所有必要的目标：

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android
```

**Section sources**
- [app/build.gradle](file://app/android/app/build.gradle#L95-L102)
- [Cargo.toml](file://app/rust/Cargo.toml#L5-L6)

## 构建缓存问题

构建缓存问题可能导致增量构建失败或使用了过时的编译结果。本项目使用Cargokit进行Rust代码的构建，其缓存机制通过`run_build_tool.sh`脚本实现：

```bash
# Dart run不会缓存任何具有路径依赖的包，因此我们自行预编译包
# 为了使缓存的内核失效，我们使用build_tool包目录的ls -LR哈希值
# 这应该足够好，因为build_tool包本身不打算有路径依赖

if [[ "$OSTYPE" == "darwin"* ]]; then
  PACKAGE_HASH=$(ls -lTR "$BUILD_TOOL_PKG_DIR" | shasum)
else
  PACKAGE_HASH=$(ls -lR --full-time "$BUILD_TOOL_PKG_DIR" | shasum)
fi
```

处理构建缓存问题的策略包括：

1. **清理构建缓存**：删除`CARGOKIT_TOOL_TEMP_DIR`目录下的缓存文件
2. **强制重新构建**：设置环境变量`CARGO_ENCODED_RUSTFLAGS`以改变编译标志
3. **检查哈希值**：确保`PACKAGE_HASH_FILE`中的哈希值与当前代码状态匹配

对于增量构建故障，可以：
- 检查`build_tool_runner.dill`文件是否是最新的
- 确认`pubspec.yaml`中的依赖没有变化
- 验证`build_tool`包的内容是否已更新

当遇到"invalid snapshot version"错误（退出码253）时，脚本会自动重新获取依赖并重新编译：

```bash
if [ $exit_code == 253 ]; then
  "$DART" pub get --no-precompile
  "$DART" compile kernel bin/build_tool_runner.dart
  "$DART" bin/build_tool_runner.dill "$@"
  exit_code=$?
fi
```

**Section sources**
- [run_build_tool.sh](file://app/rust_builder/cargokit/run_build_tool.sh#L70-L94)
- [builder.dart](file://app/rust_builder/cargokit/build_tool/lib/src/builder.dart#L82-L130)

## JNI调用异常与符号未找到

JNI调用异常和符号未找到问题通常与Rust代码的导出符号有关。在本项目中，Rust库的配置如下：

```toml
[lib]
crate-type = ["cdylib", "staticlib"]
```

`cdylib`类型确保生成动态链接库，这对于JNI调用是必需的。符号未找到的常见原因包括：

1. **函数未正确导出**：确保Rust函数使用`#[no_mangle]`和`pub extern "C"`声明
2. **名称修饰问题**：JNI需要确切的函数名称，不能有Rust的名称修饰
3. **架构不匹配**：加载的so文件架构与设备不匹配

在Flutter-Rust-Bridge集成中，这些问题通常由代码生成工具处理。检查`app/lib/rust/api/`目录下的生成代码，确保JNI接口正确生成。

调试JNI调用异常的方法：
- 使用`nm -D librust_lib_localsend_app.so`检查导出的符号
- 确认Java/Kotlin代码中的`System.loadLibrary`调用正确
- 验证JNI函数签名与Rust导出函数匹配

**Section sources**
- [Cargo.toml](file://app/rust/Cargo.toml#L5-L6)
- [pubspec.yaml](file://app/pubspec.yaml#L118-L120)

## 运行时崩溃分析

运行时崩溃可能由多种因素引起，包括内存访问错误、未处理的panic和资源泄漏。本项目使用了以下机制来帮助诊断运行时问题：

1. **错误处理框架**：使用`anyhow`和`thiserror`进行错误传播和处理
2. **日志记录**：集成`tracing`和`tracing-subscriber`进行详细的运行时日志记录
3. **异常安全**：通过`catch_unwind`等机制防止Rust panic导致整个应用崩溃

对于NDK相关的运行时崩溃，特别需要注意：
- **libgcc兼容性问题**：NDK 23及以上版本移除了libgcc，需要特殊处理：

```dart
// NDK23及更高版本的libgcc变通方案
String _libGccWorkaround(String buildDir, Version ndkVersion) {
  final workaroundDir = path.join(
    buildDir,
    'cargokit',
    'libgcc_workaround',
    '${ndkVersion.major}',
  );
  Directory(workaroundDir).createSync(recursive: true);
  if (ndkVersion.major >= 23) {
    File(path.join(workaroundDir, 'libgcc.a'))
        .writeAsStringSync('INPUT(-lunwind)');
  }
}
```

此代码在构建时为NDK 23+创建一个`libgcc.a`文件，该文件实际上是`libunwind`的符号链接，解决了链接器找不到`libgcc`的问题。

**Section sources**
- [android_environment.dart](file://app/rust_builder/cargokit/build_tool/lib/src/android_environment.dart#L168-L194)
- [Cargo.toml](file://app/rust/Cargo.toml#L10-L11)

## 日志分析工具使用

有效的日志分析是诊断Android应用问题的关键。本项目提供了多种日志分析方法：

### 使用logcat

收集Rust代码的日志输出：

```bash
adb logcat | grep localsend
```

或者针对特定严重级别：

```bash
adb logcat *:E
```

### 使用ndk-stack

当遇到native崩溃时，使用ndk-stack工具解析堆栈跟踪：

```bash
adb logcat | ndk-stack -sym ./app/build/intermediates/merged_native_libs/debug/out/lib
```

这将把混淆的堆栈跟踪转换为可读的函数名和行号。

### 自定义构建脚本

项目中的`scripts/compile_android_apk.sh`脚本为可重现构建提供了框架：

```bash
# 此APK脚本以符合F-Droid"可重现构建"的方式编写
# 可通过"readelf --wide --notes libapp.so"检查构建ID
```

通过分析生成的so文件的构建ID，可以验证构建的可重现性。

**Section sources**
- [compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L31)
- [android_environment.dart](file://app/rust_builder/cargokit/build_tool/lib/src/android_environment.dart#L98-L134)

## 结论
本文档详细介绍了在Android平台上集成Rust代码时可能遇到的常见问题及其解决方案。通过正确配置NDK版本、确保目标架构完整性、妥善处理构建缓存、正确导出JNI符号以及有效使用日志分析工具，开发者可以显著提高开发效率和应用稳定性。建议在开发过程中定期验证这些配置，以确保构建过程的可靠性和可重现性。