# WebRTC连接管理

<cite>
**本文档中引用的文件**  
- [webrtc.rs](file://core/src/webrtc/webrtc.rs)
- [signaling.rs](file://core/src/webrtc/signaling.rs)
- [webrtc_receiver.dart](file://app/lib/provider/network/webrtc/webrtc_receiver.dart)
- [webrtc.freezed.dart](file://app/lib/rust/api/webrtc.freezed.dart)
- [signaling_provider.dart](file://app/lib/provider/network/webrtc/signaling_provider.dart)
</cite>

## 目录
1. [简介](#简介)
2. [SDP交换流程](#sdp交换流程)
3. [ICE候选者收集机制](#ice候选者收集机制)
4. [连接状态机](#连接状态机)
5. [连接事件处理与监控](#连接事件处理与监控)
6. [NAT穿透实现策略](#nat穿透实现策略)
7. [连接恢复与断线重连](#连接恢复与断线重连)

## 简介
本文档详细阐述了LocalSend项目中WebRTC连接管理的核心机制。系统通过WebRTC技术实现点对点文件传输，采用SDP协议进行会话描述交换，利用STUN服务器辅助ICE候选者收集以实现NAT穿透，并通过WebSocket信令服务器协调连接建立过程。整个连接流程包含严格的加密验证和PIN码安全机制，确保传输的安全性和可靠性。

## SDP交换流程
SDP（会话描述协议）交换是WebRTC连接建立的核心步骤，用于协商媒体能力和网络参数。该流程分为发起方（offerer）和接收方（answerer）两个角色。

**发起方流程**：
1. 创建`RTCPeerConnection`实例并配置STUN服务器
2. 调用`create_offer()`生成本地SDP offer
3. 通过`set_local_description()`设置本地描述
4. 等待ICE候选者收集完成
5. 将压缩编码后的SDP通过信令服务器发送给目标设备
6. 等待并接收对方的SDP answer
7. 通过`set_remote_description()`设置远程描述完成协商

**接收方流程**：
1. 接收来自信令服务器的SDP offer
2. 通过`set_remote_description()`设置远程描述
3. 调用`create_answer()`生成SDP answer
4. 通过`set_local_description()`设置本地描述
5. 将压缩编码后的SDP answer通过信令服务器发送回发起方

SDP内容在传输前经过zlib压缩和base64编码，以减少网络传输开销。整个交换过程通过`ManagedSignalingConnection`管理，确保消息的可靠传递。

**Section sources**
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L551-L595)
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1003-L1048)
- [signaling.rs](file://core/src/webrtc/signaling.rs#L314-L360)

## ICE候选者收集机制
ICE（交互式连接建立）候选者收集是WebRTC实现NAT穿透的关键过程。系统通过STUN服务器获取不同类型的网络地址候选者。

**候选者类型**：
- **主机候选者**：设备本地网络接口的IP地址和端口
- **服务器反射候选者**：通过STUN服务器反射得到的公网IP和端口
- **中继候选者**：通过TURN服务器中继的连接（本系统未使用）

**收集流程**：
1. 创建`RTCPeerConnection`时配置STUN服务器列表
2. 系统自动开始收集候选者
3. 通过`gathering_complete_promise()`等待收集完成
4. 收集到的候选者信息包含在SDP描述中进行交换

在LocalSend中，默认配置了Google的公共STUN服务器`stun:stun.l.google.com:19302`，同时允许用户自定义STUN服务器配置。候选者收集完成后，系统会进行P2P连接尝试，优先选择延迟最低的路径。

**Section sources**
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1050-L1095)
- [main.rs](file://core/src/main.rs#L226)

## 连接状态机
WebRTC连接管理采用状态机模式来跟踪连接的生命周期，确保各个阶段的正确转换。

**状态定义**：
- **new**：连接初始状态，尚未开始SDP交换
- **connecting**：正在进行SDP交换和ICE协商
- **connected**：数据通道已打开，可以进行通信
- **disconnected**：连接已断开，可能需要重连
- **finished**：文件传输完成，连接正常关闭
- **error**：发生错误，连接终止

**状态转换条件**：
- `new` → `connecting`：开始SDP offer流程
- `connecting` → `connected`：成功完成SDP交换和PIN验证
- `connected` → `sending`：开始文件传输
- `sending` → `finished`：所有文件传输完成
- 任意状态 → `error`：发生网络错误或验证失败
- 任意状态 → `disconnected`：对端主动关闭连接

状态变化通过`mpsc::Sender<RTCStatus>`通道通知上层应用，UI界面根据状态变化更新显示。

**Section sources**
- [webrtc.freezed.dart](file://app/lib/rust/api/webrtc.freezed.dart#L16-L55)
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1185-L1232)

## 连接事件处理与监控
系统提供了完整的连接事件处理机制和监控功能，确保传输过程的可视化和可调试性。

**事件处理**：
- **状态监听**：通过`listenStatus()`方法监听连接状态变化
- **文件列表接收**：通过`listenFiles()`获取待接收的文件列表
- **错误处理**：通过专门的错误通道接收传输过程中的错误信息
- **PIN码处理**：通过PIN码通道处理安全验证请求

**统计信息收集**：
- 连接建立时间
- ICE候选者收集时间
- 数据通道打开时间
- 文件传输速度和进度
- 网络延迟和丢包率

**连接质量监控**：
系统通过定期检查数据通道的缓冲区状态（`buffered_amount()`）来监控连接质量。当缓冲区数据量持续较高时，可能表示网络拥塞或接收方处理能力不足。此外，通过`wait_buffer_empty()`确保所有数据都已成功发送后再关闭连接。

**Section sources**
- [webrtc_receiver.dart](file://app/lib/provider/network/webrtc/webrtc_receiver.dart#L0-L218)
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1050-L1095)

## NAT穿透实现策略
在本地网络环境下，系统采用多种策略实现NAT穿透，确保不同网络环境下的设备能够成功建立P2P连接。

**ICE打洞技术**：
1. **STUN服务器探测**：使用STUN服务器确定设备的公网映射地址
2. **候选者配对**：将本地候选者与远程候选者进行配对测试
3. **连接性检查**：通过STUN Binding请求验证候选者对的连通性
4. **路径选择**：选择延迟最低、最可靠的连接路径

**连接优先级排序**：
系统按照以下优先级顺序尝试建立连接：
1. **主机候选者**：直接在同一局域网内的设备间通信，延迟最低
2. **服务器反射候选者**：通过STUN服务器反射的公网地址，适用于不同NAT类型
3. **中继候选者**：作为最后手段，通过TURN服务器中继（当前未实现）

**实现特点**：
- 使用Google公共STUN服务器作为默认配置
- 支持自定义STUN服务器列表
- 采用UDP协议进行ICE候选者探测
- 实现了完整的ICE控制流程，包括收集、配对和检查阶段

**Section sources**
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1050-L1095)
- [main.rs](file://core/src/main.rs#L226)

## 连接恢复与断线重连
系统实现了稳健的连接恢复机制和断线重连策略，确保在网络不稳定情况下的传输可靠性。

**连接恢复机制**：
- **自动重试**：在网络中断后自动尝试重新建立连接
- **状态保持**：保存传输进度，恢复后从中断点继续
- **超时控制**：设置合理的重连超时时间，避免无限重试
- **错误分类**：区分临时性错误和永久性错误，采取不同处理策略

**断线重连策略**：
1. **检测断开**：通过`on_peer_connection_state_change`监听连接状态变化
2. **快速重连**：在短时间内尝试重新建立连接
3. **退避算法**：采用指数退避策略，避免频繁重试造成网络拥塞
4. **资源清理**：在重连前清理旧的连接资源，防止内存泄漏

当连接状态变为`Disconnected`时，系统会触发重连流程。如果多次重连失败，则进入`Error`状态并通知用户。对于文件传输任务，系统会保存已传输的数据，恢复连接后继续未完成的传输。

**Section sources**
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1050-L1095)
- [webrtc.rs](file://core/src/webrtc/webrtc.rs#L1185-L1232)