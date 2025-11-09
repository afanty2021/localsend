# Android 实现

<cite>
**本文档中引用的文件**  
- [AndroidManifest.xml](file://app/android/app/src/main/AndroidManifest.xml)
- [build.gradle](file://app/android/app/build.gradle)
- [android_channel.dart](file://app/lib/util/native/channel/android_channel.dart)
- [MainActivity.kt](file://app/android/app/src/main/kotlin/org/localsend/localsend_app/MainActivity.kt)
- [content_uri_helper.dart](file://app/lib/util/native/content_uri_helper.dart)
- [compile_android_apk.sh](file://scripts/compile_android_apk.sh)
- [compile_android_appbundle.ps1](file://scripts/compile_android_appbundle.ps1)
</cite>

## 目录
1. [AndroidManifest.xml权限配置](#androidmanifestxml权限配置)
2. [build.gradle构建配置](#buildgradle构建配置)
3. [Dart与Android原生通信](#dart与android原生通信)
4. [Android特定功能实现](#android特定功能实现)
5. [Android调试技巧](#android调试技巧)
6. [APK和App Bundle构建指南](#apk和app-bundle构建指南)
7. [Google Play发布注意事项](#google-play发布注意事项)

## AndroidManifest.xml权限配置

在`AndroidManifest.xml`文件中，项目配置了多种Android权限以支持应用功能。网络访问权限`android.permission.INTERNET`允许应用进行网络通信。文件读写权限包括`READ_EXTERNAL_STORAGE`和`WRITE_EXTERNAL_STORAGE`，以及针对Android 10及以上版本的`READ_MEDIA_IMAGES`和`READ_MEDIA_VIDEO`，这些权限使应用能够访问用户的媒体文件。

应用还请求了`ACCESS_MEDIA_LOCATION`权限以访问媒体文件的位置信息，以及`QUERY_ALL_PACKAGES`和`REQUEST_INSTALL_PACKAGES`权限以支持应用安装包的查询和安装功能。此外，清单文件中配置了`android:usesCleartextTraffic="true"`以允许明文流量，这对于本地网络通信是必要的。

**Section sources**
- [AndroidManifest.xml](file://app/android/app/src/main/AndroidManifest.xml#L1-L75)

## build.gradle构建配置

`build.gradle`文件配置了Android应用的构建参数。应用使用了`com.android.application`和`kotlin-android`插件，并设置了`compileSdkVersion`为34。构建配置中指定了Java和Kotlin的兼容性版本为17，这确保了对现代Java特性的支持。

在`defaultConfig`中，应用ID设置为`org.localsend.localsend_app`，并从`local.properties`文件中读取版本代码和版本名称。构建类型包括`release`和`debug`，其中`release`构建使用`signingConfigs.release`进行签名配置，从`key.properties`文件中读取签名信息。

为了兼容F-Droid构建要求，配置了`dependenciesInfo`来禁用APK和App Bundle中的依赖元数据。此外，通过`abiCodes`映射为不同ABI（x86_64、armeabi-v7a、arm64-v8a）分配了版本代码偏移量，以符合F-Droid的版本代码方案。

**Section sources**
- [build.gradle](file://app/android/app/build.gradle#L1-L103)

## Dart与Android原生通信

通过`android_channel.dart`文件实现的Dart与Android原生代码通信机制基于Flutter的MethodChannel。该机制定义了一个名为`org.localsend.localsend_app/localsend`的方法通道，用于在Dart代码和Android原生代码之间传递方法调用和结果。

主要通信方法包括`pickDirectory`、`pickFiles`和`pickDirectoryPath`，这些方法用于触发Android的文件选择器并返回所选文件或目录的URI。`createDirectory`方法允许在指定的文档URI下创建新目录，而`openContentUri`和`openGallery`方法则用于打开内容URI和系统图库。

在Android原生端，`MainActivity.kt`中的`configureFlutterEngine`方法设置了方法调用处理器，根据接收到的方法名调用相应的原生功能。例如，`pickDirectory`方法会启动`ACTION_OPEN_DOCUMENT_TREE`意图来选择目录，而`pickFiles`方法则使用`ACTION_OPEN_DOCUMENT`意图来选择文件。

**Section sources**
- [android_channel.dart](file://app/lib/util/native/channel/android_channel.dart#L1-L120)
- [MainActivity.kt](file://app/android/app/src/main/kotlin/org/localsend/localsend_app/MainActivity.kt#L1-L290)

## Android特定功能实现

### 文件访问权限处理

对于Android 10及以上版本，应用使用存储访问框架（SAF）来处理文件访问权限。`content_uri_helper.dart`文件提供了辅助函数来处理内容URI，包括从树URI提取路径、猜测相对路径以及将树URI转换为文档URI。这些函数支持应用在用户授予持久性URI权限后，对选定目录进行递归文件操作。

### 后台服务运行

应用通过配置`android:launchMode="singleTask"`来防止多个实例同时运行，确保应用在后台运行时能够正确处理分享意图。当应用被其他应用通过分享功能调用时，系统会将现有实例带到前台而不是创建新实例。

### 通知系统集成

虽然在提供的代码中没有直接看到通知系统的实现，但应用通过`share_handler`依赖项集成了分享功能，允许用户从其他应用分享文件到LocalSend。当接收到分享意图时，应用会启动并处理传入的文件。

### 上下文菜单支持

在Windows平台上，应用通过`context_menu_helper.dart`文件实现了上下文菜单集成。该功能允许用户在文件资源管理器的右键菜单中直接使用LocalSend发送文件。通过PowerShell脚本创建快捷方式并放置在"SendTo"文件夹中，实现了这一功能。

**Section sources**
- [content_uri_helper.dart](file://app/lib/util/native/content_uri_helper.dart#L1-L106)
- [MainActivity.kt](file://app/android/app/src/main/kotlin/org/localsend/localsend_app/MainActivity.kt#L1-L290)
- [context_menu_helper.dart](file://app/lib/util/native/context_menu_helper.dart#L1-L65)

## Android调试技巧

### 权限拒绝处理

当用户拒绝文件访问权限时，应用通过`file_picker.dart`中的`_pickFolder`方法处理权限请求。该方法在Android平台上显式请求`Permission.storage`权限，并在用户拒绝后显示`NoPermissionDialog`提示用户手动授予权限。

### 文件访问问题排查

对于文件访问问题，应用使用`content_uri_helper.dart`中的`guessRelativePathFromPickedFileContentUri`函数来从内容URI猜测相对路径。此外，`file_saver.dart`中的`digestFilePathAndPrepareDirectory`函数处理了SD卡路径的特殊情况，并确保在创建目录时防止路径遍历攻击。

在调试过程中，可以使用`DebugPage`中的各种调试入口来检查应用状态、HTTP日志和安全设置。这些调试工具帮助开发者识别和解决文件访问和网络通信中的问题。

**Section sources**
- [file_picker.dart](file://app/lib/util/native/file_picker.dart#L1-L197)
- [file_saver.dart](file://app/lib/util/native/file_saver.dart#L1-L231)
- [debug_page.dart](file://app/lib/pages/debug/debug_page.dart#L1-L23)

## APK和App Bundle构建指南

### 构建APK

通过`compile_android_apk.sh`脚本可以构建APK文件。该脚本首先清理临时目录，然后复制项目到临时位置进行构建。它使用子模块中的Flutter版本，执行`flutter pub get`获取依赖，运行`build_runner`生成代码，最后使用`flutter build apk`命令生成APK文件。该脚本设计为符合F-Droid的可重现构建要求。

### 构建App Bundle

通过`compile_android_appbundle.ps1`脚本可以构建App Bundle。该脚本使用FVM（Flutter Version Management）来确保使用正确的Flutter版本。它执行`flutter clean`清理构建缓存，获取依赖，然后使用`flutter build appbundle`命令生成AAB文件。PowerShell脚本的使用表明该构建过程主要针对Windows开发环境。

两种构建方式都支持从最新提交构建，只需取消脚本中的git重置和拉取命令注释即可。构建过程考虑了签名配置，从`key.properties`文件中读取签名信息用于发布版本。

**Section sources**
- [compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)
- [compile_android_appbundle.ps1](file://scripts/compile_android_appbundle.ps1#L1-L11)

## Google Play发布注意事项

在发布到Google Play时，需要注意几个关键事项。首先，应用使用了`targetSdkVersion`为34，这符合Google Play对新应用的最新要求。其次，应用请求了`QUERY_ALL_PACKAGES`权限，这属于敏感权限，需要在Play Console中提供详细的使用理由和用户数据保护措施。

由于应用涉及文件共享功能，需要确保遵守Google Play的用户数据政策，特别是关于文件访问和隐私保护的规定。应用的权限使用应该遵循最小权限原则，只请求必要的权限，并在用户界面中清晰地解释权限用途。

对于应用内购买功能（在`pubspec.yaml`中引用了`in_app_purchase`），如果使用了该功能，需要确保遵守Google Play的计费政策。此外，应用的图标、截图和描述需要符合Google Play的审核指南，避免使用误导性或夸张的表述。

**Section sources**
- [build.gradle](file://app/android/app/build.gradle#L1-L103)
- [pubspec.yaml](file://app/pubspec.yaml#L1-L124)