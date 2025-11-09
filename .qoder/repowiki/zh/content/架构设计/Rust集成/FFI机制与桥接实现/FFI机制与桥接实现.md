# FFI机制与桥接实现

<cite>
**本文档引用的文件**
- [flutter_rust_bridge.yaml](file://app/flutter_rust_bridge.yaml)
- [lib.rs](file://app/rust/src/lib.rs)
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart)
- [frb_generated.rs](file://app/rust/src/frb_generated.rs)
- [model.rs](file://app/rust/src/api/model.rs)
- [webrtc.rs](file://app/rust/src/api/webrtc.rs)
- [crypto.rs](file://app/rust/src/api/crypto.rs)
</cite>

## 目录
1. [介绍](#介绍)
2. [FFI机制概述](#ffi机制概述)
3. [Flutter-Rust-Bridge配置](#flutter-rust-bridge配置)
4. [桥接代码生成机制](#桥接代码生成机制)
5. [数据类型映射规则](#数据类型映射规则)
6. [同步与异步函数生成模式](#同步与异步函数生成模式)
7. [桥接调用流程](#桥接调用流程)
8. [内存管理与生命周期](#内存管理与生命周期)
9. [错误处理机制](#错误处理机制)
10. [调试技巧](#调试技巧)
11. [实际应用示例](#实际应用示例)

## 介绍

LocalSend项目使用Flutter-Rust-Bridge（FRB）实现Dart与Rust之间的高效互操作。本文档详细分析FFI机制与桥接实现，涵盖配置、代码生成、数据类型映射、调用流程等核心方面。

**Section sources**
- [lib.rs](file://app/rust/src/lib.rs#L1-L4)

## FFI机制概述

FFI（Foreign Function Interface）机制允许Dart代码调用Rust编写的高性能函数。LocalSend通过FRB实现这一机制，将Rust的WebRTC、加密等核心功能暴露给Dart前端。

FRB在Dart与Rust之间创建了一层抽象，自动处理数据序列化、内存管理和线程调度。这种设计既保证了性能，又简化了开发复杂性。

**Section sources**
- [lib.rs](file://app/rust/src/lib.rs#L1-L4)

## Flutter-Rust-Bridge配置

`flutter_rust_bridge.yaml`文件定义了FRB的核心配置：

```yaml
rust_input: crate::api
rust_root: rust/
dart_output: lib/rust
```

该配置指定了：
- `rust_input`: Rust API的入口模块（`crate::api`）
- `rust_root`: Rust代码的根目录
- `dart_output`: 生成的Dart代码输出目录

此配置指导FRB工具从`api`模块生成桥接代码到`lib/rust`目录。

**Section sources**
- [flutter_rust_bridge.yaml](file://app/flutter_rust_bridge.yaml#L1-L4)

## 桥接代码生成机制

FRB根据Rust代码自动生成桥接文件，主要包括：

1. **Dart侧**: `frb_generated.dart` - Dart API入口
2. **Rust侧**: `frb_generated.rs` - Rust FFI绑定

生成过程基于`#[frb]`属性宏标记的类型和函数。例如，在`model.rs`中：

```rust
#[frb(mirror(RegisterDto))]
pub struct _RegisterDto {
    pub alias: String,
    pub version: String,
    // ... 其他字段
}
```

`mirror`属性指示FRB为`RegisterDto`类型生成双向映射代码。

**Section sources**
- [model.rs](file://app/rust/src/api/model.rs#L5-L15)
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L1-L80)
- [frb_generated.rs](file://app/rust/src/frb_generated.rs#L1-L100)

## 数据类型映射规则

### 基本数据类型映射

FRB定义了Dart与Rust基本类型之间的直接映射：

| Dart类型 | Rust类型 | 序列化方法 |
|---------|---------|----------|
| `String` | `String` | UTF-8编码 |
| `int` | `i32`/`u32` | 32位整数 |
| `BigInt` | `u64`/`i64` | 64位整数 |
| `bool` | `bool` | 单字节 |
| `List<int>` | `Vec<u8>` | 字节数组 |

### 枚举类型映射

枚举类型通过标签（tag）进行序列化。以`RTCStatus`为例：

```dart
@protected
void sse_encode_rtc_status(RTCStatus self, SseSerializer serializer) {
    switch (self) {
      case RTCStatus_SdpExchanged():
        sse_encode_i_32(0, serializer);
      case RTCStatus_Connected():
        sse_encode_i_32(1, serializer);
      // ... 其他状态
      case RTCStatus_Error(field0: final field0):
        sse_encode_i_32(7, serializer);
        sse_encode_String(field0, serializer);
    }
}
```

枚举值首先序列化其标签（0-7），对于包含数据的变体（如`Error`），随后序列化其数据。

**Section sources**
- [webrtc.rs](file://app/rust/src/api/webrtc.rs#L400-L420)
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L2777-L2828)
- [frb_generated.rs](file://app/rust/src/frb_generated.rs#L2011-L2042)

### 结构体与复杂对象映射

结构体字段按声明顺序逐个序列化。以`FileDto`为例：

```dart
@protected
void sse_encode_file_dto(FileDto self, SseSerializer serializer) {
    sse_encode_String(self.id, serializer);
    sse_encode_String(self.fileName, serializer);
    sse_encode_u_64(self.size, serializer);
    sse_encode_String(self.fileType, serializer);
    sse_encode_opt_String(self.sha256, serializer);
    sse_encode_opt_String(self.preview, serializer);
    sse_encode_opt_box_autoadd_file_metadata(self.metadata, serializer);
}
```

嵌套对象（如`FileMetadata`）递归序列化，可选字段通过特殊标记处理。

**Section sources**
- [model.rs](file://app/rust/src/api/model.rs#L40-L55)
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L2607-L2643)

## 同步与异步函数生成模式

### 同步函数

同步函数直接返回结果值。例如密钥对生成：

```dart
Future<KeyPair> crateApiCryptoGenerateKeyPair();
```

尽管使用`Future`，但这类函数在Rust侧是同步执行的，通过FFI调用后立即返回。

### 异步函数

异步函数使用`async/await`模式。FRB生成的代码通过端口（port）机制处理异步操作：

```dart
Future<RtcReceiveController> crateApiWebrtcLsSignalingConnectionAcceptOffer(
    {required LsSignalingConnection that,
    required List<String> stunServers,
    required WsServerSdpMessage offer,
    required String privateKey,
    ExpectingPublicKey? expectingPublicKey,
    PinConfig? pin}) {
  return handler.executeNormal(NormalTask(
    callFfi: (port_) {
      // 序列化参数并调用FFI
      pdeCallFfi(generalizedFrbRustBinding, serializer, funcId: 1, port: port_);
    },
    codec: SseCodec(
      decodeSuccessData: sse_decode_Auto_Owned_RustOpaque_flutter_rust_bridgefor_generatedRustAutoOpaqueInnerRTCReceiveController,
    ),
    // ... 其他配置
  ));
}
```

`port_`参数用于异步通信，Rust侧完成操作后通过该端口返回结果。

### 流式函数

流式函数（返回`Stream`）使用`StreamSink`机制：

```dart
Stream<RTCStatus> crateApiWebrtcRtcReceiveControllerListenStatus(
    {required RtcReceiveController that}) {
  final sink = RustStreamSink<RTCStatus>();
  unawaited(handler.executeNormal(NormalTask(
    callFfi: (port_) {
      sse_encode_Auto_Ref_RustOpaque_flutter_rust_bridgefor_generatedRustAutoOpaqueInnerRTCReceiveController(that, serializer);
      sse_encode_StreamSink_rtc_status_Sse(sink, serializer);
      pdeCallFfi(generalizedFrbRustBinding, serializer, funcId: 11, port: port_);
    },
    // ... 其他配置
  )));
  return sink.stream;
}
```

`sink`对象在Rust侧被用来推送流式数据，Dart侧通过`stream`属性接收。

**Section sources**
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L100-L200)
- [webrtc.rs](file://app/rust/src/api/webrtc.rs#L100-L150)

## 桥接调用流程

完整的桥接调用流程包括以下步骤：

1. **参数序列化**: Dart参数通过SSE编码器序列化为字节流
2. **FFI调用**: 调用原生函数，传递序列化数据和端口
3. **Rust侧反序列化**: Rust代码反序列化参数
4. **函数执行**: 执行实际的Rust函数
5. **结果序列化**: 将结果序列化回字节流
6. **Dart侧反序列化**: Dart代码反序列化结果

对于异步操作，流程通过端口机制实现非阻塞通信。

**Section sources**
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L100-L300)
- [frb_generated.rs](file://app/rust/src/frb_generated.rs#L1000-L1100)

## 内存管理与生命周期

### 引用计数管理

Rust的`Arc`（原子引用计数）类型通过专门的函数进行管理：

```dart
RustArcIncrementStrongCountFnType
    get rust_arc_increment_strong_count_LsSignalingConnection;

RustArcDecrementStrongCountFnType
    get rust_arc_decrement_strong_count_LsSignalingConnection;
```

这些函数暴露给Dart侧，用于手动管理Rust对象的生命周期。

### 自动内存管理

大多数情况下，FRB通过`RustOpaque`包装器实现自动内存管理。当Dart对象被垃圾回收时，对应的Rust对象引用计数会自动递减。

```dart
class RtcFileReceiverImpl extends RustOpaque implements RtcFileReceiver {
  // Not to be used by end users
  RtcFileReceiverImpl.frbInternalDcoDecode(List<dynamic> wire)
      : super.frbInternalDcoDecode(wire, _kStaticData);
}
```

`RustOpaque`基类确保了适当的资源清理。

**Section sources**
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L80-L100)
- [frb_generated.io.dart](file://app/lib/rust/frb_generated.io.dart#L852-L866)

## 错误处理机制

### 异常传播

Rust的`anyhow::Result`类型映射到Dart的`Future`错误：

```dart
codec: SseCodec(
  decodeSuccessData: sse_decode_Auto_Owned_RustOpaque_flutter_rust_bridgefor_generatedRustAutoOpaqueInnerRTCReceiveController,
  decodeErrorData: sse_decode_AnyhowException,
)
```

成功结果通过`decodeSuccessData`处理，错误通过`decodeErrorData`处理。

### 错误类型映射

自定义错误类型如`RTCFileError`被完整映射：

```rust
#[frb(mirror(RTCFileError))]
pub struct _RTCFileError {
    pub file_id: String,
    pub error: String,
}
```

这确保了错误信息在跨语言调用中保持完整。

**Section sources**
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L150-L180)
- [webrtc.rs](file://app/rust/src/api/webrtc.rs#L410-L415)

## 调试技巧

### 日志记录

在`webrtc.rs`中，连接函数包含详细的错误处理：

```rust
pub async fn connect(
    sink: StreamSink<WsServerMessage>,
    uri: String,
    info: ProposingClientInfo,
    private_key: String,
    on_connection: impl Fn(LsSignalingConnection) -> DartFnFuture<()>,
) {
    let Ok(signing_key) = localsend::crypto::token::parse_private_key(&private_key) else {
        let _ = sink.add_error(anyhow::anyhow!("Invalid private key"));
        return;
    };
    // ... 其他代码
}
```

使用`sink.add_error()`可以将错误信息传递回Dart侧进行日志记录。

### 调试策略

1. **启用调试日志**: 调用`crateApiLoggingEnableDebugLogging()`启用详细日志
2. **检查序列化**: 验证复杂对象的序列化/反序列化过程
3. **监控内存**: 注意`Arc`对象的引用计数变化
4. **异步跟踪**: 使用端口ID跟踪异步调用的完整生命周期

**Section sources**
- [webrtc.rs](file://app/rust/src/api/webrtc.rs#L50-L100)
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L90-L95)

## 实际应用示例

### 密钥对生成

```dart
Future<KeyPair> generateKeyPair() async {
  await RustLib.init(); // 初始化FRB
  return await RustLib.instance.api.crateApiCryptoGenerateKeyPair();
}
```

### WebRTC连接

```dart
Future<RtcSendController> sendOffer(
  LsSignalingConnection connection,
  List<String> stunServers,
  UuidValue target,
  String privateKey,
  List<FileDto> files,
) async {
  return await connection.sendOffer(
    stunServers: stunServers,
    target: target,
    privateKey: privateKey,
    files: files,
  );
}
```

### 流式状态监听

```dart
Stream<RTCStatus> listenStatus(RtcSendController controller) {
  return controller.listenStatus();
}
```

这些示例展示了如何在实际应用中使用生成的桥接API。

**Section sources**
- [frb_generated.dart](file://app/lib/rust/frb_generated.dart#L2891-L2930)
- [webrtc.rs](file://app/rust/src/api/webrtc.rs#L150-L200)