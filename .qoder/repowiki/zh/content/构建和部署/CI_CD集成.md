# CI/CD集成

<cite>
**本文档中引用的文件**  
- [compile_android_apk.sh](file://scripts/compile_android_apk.sh)
- [compile_android_appbundle.ps1](file://scripts/compile_android_appbundle.ps1)
- [compile_ios.sh](file://scripts/compile_ios.sh)
- [compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1)
- [compile_windows_msix_store.ps1](file://scripts/compile_windows_msix_store.ps1)
- [Dockerfile](file://server/Dockerfile)
- [pubspec.yaml](file://app/pubspec.yaml)
- [build.yaml](file://app/build.yaml)
- [AppImageBuilder_x86_64.yml](file://scripts/appimage/AppImageBuilder_x86_64.yml)
- [AppImageBuilder_arm_64.yml](file://scripts/appimage/AppImageBuilder_arm_64.yml)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概述](#架构概述)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介
LocalSend是一个开源的跨平台文件传输应用，提供类似AirDrop的功能。本CI/CD集成文档详细介绍了该项目的持续集成和持续部署流水线配置，包括自动化构建、测试、签名和发布流程。文档涵盖了多个平台的发布要求，包括Android、iOS、Windows和Linux，以及Docker容器化部署的配置。

## 项目结构
LocalSend项目采用多平台架构，包含Flutter前端、Rust后端和独立的CLI工具。项目结构清晰地分离了不同平台的构建配置和资源文件。

```mermaid
graph TD
A[项目根目录] --> B[app]
A --> C[cli]
A --> D[common]
A --> E[core]
A --> F[server]
A --> G[scripts]
A --> H[fastlane]
A --> I[msix]
B --> B1[android]
B --> B2[ios]
B --> B3[linux]
B --> B4[macos]
B --> B5[windows]
B --> B6[lib]
B --> B7[assets]
F --> F1[src]
F --> F2[Dockerfile]
F --> F3[Cargo.toml]
G --> G1[appimage]
G --> G2[msix]
G --> G3[*.sh]
G --> G4[*.ps1]
```

**Diagram sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh)
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh)
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1)

**Section sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh#L1-L14)
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1#L1-L20)

## 核心组件
LocalSend的核心组件包括Flutter应用前端、Rust后端服务器和共享的公共库。CI/CD系统需要协调这些组件的构建和集成。

**Section sources**
- [app/pubspec.yaml](file://app/pubspec.yaml#L1-L124)
- [server/Cargo.toml](file://server/Cargo.toml#L1-L19)
- [common/pubspec.yaml](file://common/pubspec.yaml#L1-L20)

## 架构概述
LocalSend的CI/CD架构采用多平台并行构建策略，每个目标平台都有专门的构建脚本。系统通过Shell和PowerShell脚本实现跨平台构建自动化。

```mermaid
graph TB
subgraph "构建触发"
A[代码提交]
B[手动触发]
end
subgraph "构建环境"
C[Linux/Android]
D[macOS/iOS]
E[Windows]
end
subgraph "构建流程"
F[清理工作区]
G[获取依赖]
H[编译代码]
I[生成包]
J[签名]
K[发布]
end
A --> F
B --> F
F --> C
F --> D
F --> E
C --> G
D --> G
E --> G
G --> H
H --> I
I --> J
J --> K
```

**Diagram sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh#L1-L14)
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1#L1-L20)

## 详细组件分析

### Android构建流程
Android构建流程设计为符合F-Droid的可重现构建要求，使用特定的SDK路径和构建参数确保构建的一致性。

```mermaid
flowchart TD
A["开始Android构建"] --> B["设置环境: JDK, SDKManager"]
B --> C["安装Android平台工具"]
C --> D["接受SDK许可"]
D --> E["克隆最新代码"]
E --> F["更新Git子模块"]
F --> G["配置Flutter分析"]
G --> H["获取Dart包依赖"]
H --> I["运行构建生成器"]
I --> J["构建APK"]
J --> K["输出APK文件"]
K --> L["结束"]
style A fill:#f9f,stroke:#333
style L fill:#bbf,stroke:#333
```

**Diagram sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)

**Section sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)
- [app/android/build.gradle](file://app/android/build.gradle#L1-L35)

### iOS构建流程
iOS构建流程使用fvm（Flutter Version Management）确保Flutter版本的一致性，并通过CocoaPods管理原生依赖。

```mermaid
flowchart TD
A["开始iOS构建"] --> B["清理构建缓存"]
B --> C["获取Dart包依赖"]
C --> D["预缓存iOS构建资源"]
D --> E["进入iOS目录"]
E --> F["更新CocoaPods依赖"]
F --> G["构建IPA文件"]
G --> H["输出IPA文件"]
H --> I["结束"]
style A fill:#f9f,stroke:#333
style I fill:#bbf,stroke:#333
```

**Diagram sources**
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh#L1-L14)
- [app/ios/Podfile](file://app/ios/Podfile#L1-L49)

**Section sources**
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh#L1-L14)
- [app/ios/Podfile](file://app/ios/Podfile#L1-L49)

### Windows构建流程
Windows构建流程包括生成MSIX包，用于Microsoft Store发布，以及生成传统的EXE安装程序。

```mermaid
flowchart TD
A["开始Windows构建"] --> B["清理构建缓存"]
B --> C["获取Dart包依赖"]
C --> D["构建Windows应用"]
D --> E{"构建类型"}
E --> |MSIX| F["创建MSIX包"]
E --> |EXE| G["复制到Inno安装目录"]
G --> H["运行Inno Setup编译"]
H --> I["生成EXE安装程序"]
F --> J["重命名MSIX文件"]
I --> K["输出安装程序"]
J --> K
K --> L["结束"]
style A fill:#f9f,stroke:#333
style L fill:#bbf,stroke:#333
```

**Diagram sources**
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1#L1-L20)
- [scripts/compile_windows_msix_store.ps1](file://scripts/compile_windows_msix_store.ps1#L1-L13)

**Section sources**
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1#L1-L20)
- [scripts/compile_windows_msix_store.ps1](file://scripts/compile_windows_msix_store.ps1#L1-L13)
- [app/windows/CMakeLists.txt](file://app/windows/CMakeLists.txt#L1-L109)

### Linux构建流程
Linux构建流程使用AppImage格式，支持x86_64和ARM64架构，通过AppImageBuilder工具生成可执行文件。

**Section sources**
- [scripts/appimage/AppImageBuilder_x86_64.yml](file://scripts/appimage/AppImageBuilder_x86_64.yml)
- [scripts/appimage/AppImageBuilder_arm_64.yml](file://scripts/appimage/AppImageBuilder_arm_64.yml)

### 服务器Docker化
服务器组件通过Dockerfile进行容器化，采用多阶段构建优化镜像大小。

```mermaid
flowchart TD
A["Docker多阶段构建"] --> B["构建阶段: rust:1.83"]
B --> C["设置工作目录/app"]
C --> D["复制Cargo文件"]
D --> E["创建虚拟main.rs"]
E --> F["预编译依赖"]
F --> G["复制源代码"]
G --> H["重新编译应用"]
H --> I["运行阶段: debian:bookworm"]
I --> J["复制编译后的二进制文件"]
J --> K["暴露端口3000"]
K --> L["设置入口点"]
L --> M["完成镜像构建"]
style A fill:#f9f,stroke:#333
style M fill:#bbf,stroke:#333
```

**Diagram sources**
- [server/Dockerfile](file://server/Dockerfile#L1-L25)

**Section sources**
- [server/Dockerfile](file://server/Dockerfile#L1-L25)
- [server/Cargo.toml](file://server/Cargo.toml#L1-L19)

## 依赖分析
LocalSend项目的依赖关系复杂，涉及多个语言和平台的依赖管理。

```mermaid
graph TD
A[Flutter应用] --> B[Dart包]
A --> C[Rust库]
A --> D[原生平台依赖]
B --> B1[basic_utils]
B --> B2[flutter_rust_bridge]
B --> B3[device_info_plus]
B --> B4[permission_handler]
C --> C1[core模块]
C --> C2[common模块]
C --> C3[rust_lib_localsend_app]
D --> D1[Android: device_apps]
D --> D2[iOS: CocoaPods]
D --> D3[Windows: win32_registry]
D --> D4[Linux: desktop_drop]
E[服务器] --> F[Rust Cargo依赖]
F --> F1[axum]
F --> F2[tokio]
F --> F3[serde]
F --> F4[tracing]
G[CLI工具] --> H[Dart依赖]
H --> H1[common模块]
H --> H2[localsend_core]
```

**Diagram sources**
- [app/pubspec.yaml](file://app/pubspec.yaml#L1-L124)
- [server/Cargo.toml](file://server/Cargo.toml#L1-L19)
- [common/pubspec.yaml](file://common/pubspec.yaml#L1-L20)

**Section sources**
- [app/pubspec.yaml](file://app/pubspec.yaml#L1-L124)
- [server/Cargo.toml](file://server/Cargo.toml#L1-L19)
- [common/pubspec.yaml](file://common/pubspec.yaml#L1-L20)

## 性能考虑
CI/CD流程中的性能优化主要体现在构建缓存、并行构建和镜像优化等方面。

- **构建缓存**: 使用fvm确保Flutter版本一致性，避免重复下载
- **多阶段Docker构建**: 分离构建和运行环境，减小最终镜像大小
- **依赖预编译**: 在Docker构建中预编译Rust依赖，提高后续构建速度
- **并行构建**: 不同平台的构建可以并行执行，减少总构建时间

## 故障排除指南
CI/CD流程中常见的问题及解决方案：

**Section sources**
- [scripts/compile_android_apk.sh](file://scripts/compile_android_apk.sh#L1-L32)
- [scripts/compile_ios.sh](file://scripts/compile_ios.sh#L1-L14)
- [scripts/compile_windows_exe.ps1](file://scripts/compile_windows_exe.ps1#L1-L20)

### Android构建问题
- **问题**: SDK许可未接受
- **解决方案**: 运行`sdkmanager --licenses`并接受所有许可
- **问题**: Gradle版本冲突
- **解决方案**: 确保使用项目指定的Gradle版本

### iOS构建问题
- **问题**: CocoaPods依赖更新失败
- **解决方案**: 运行`pod repo update`更新本地仓库
- **问题**: 代码签名错误
- **解决方案**: 检查Apple开发者账户和证书配置

### Windows构建问题
- **问题**: Inno Setup未安装
- **解决方案**: 安装Inno Setup并确保在系统路径中
- **问题**: MSIX签名失败
- **解决方案**: 检查代码签名证书配置

### Docker构建问题
- **问题**: 基础镜像拉取失败
- **解决方案**: 检查网络连接和Docker Hub访问权限
- **问题**: 依赖编译失败
- **解决方案**: 确保Rust工具链版本兼容

## 结论
LocalSend项目的CI/CD系统设计完善，支持多平台构建和发布。通过Shell和PowerShell脚本实现跨平台自动化，确保了构建过程的一致性和可重现性。Docker化部署简化了服务器组件的部署流程，而详细的构建脚本提供了灵活性和可定制性。建议进一步集成GitHub Actions或GitLab CI等CI系统，实现完全自动化的持续集成和部署流程。