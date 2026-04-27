# UE5_Lyra学习指南_013_泽_01_DTLS概念.md

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_013\_泽\_01\_DTLS概念.md](#ue5_lyra学习指南_013_泽_01_dtls概念md)
  - [DTLS](#dtls)
    - [DTLS在UE中的核心作用](#dtls在ue中的核心作用)
    - [UUE中DTLS的应用场景](#uue中dtls的应用场景)
    - [相关技术](#相关技术)
    - [注意事项](#注意事项)
  - [FEncryptionData](#fencryptiondata)
    - [成员说明](#成员说明)
    - [典型应用场景](#典型应用场景)
    - [示例代码](#示例代码)
    - [注意事项](#注意事项-1)
  - [总结](#总结)



## DTLS
在UE中DTLS是什么作用?

在Unreal Engine (UE) 中，**DTLS**（Datagram Transport Layer Security）是一种基于UDP的安全通信协议，主要用于加密和验证数据传输，确保网络通信的机密性和完整性。它在UE中的作用主要与**网络通信安全**相关，尤其是在使用UDP协议的场景（如实时多人游戏、音视频传输等）。

---

### DTLS在UE中的核心作用
1. **加密UDP通信**  
   - UDP默认是无连接、不加密的协议，容易被窃听或篡改。DTLS通过TLS/SSL的加密机制为UDP提供安全保障，防止数据泄露或中间人攻击。
   - 适用于需要低延迟但需安全性的场景（如语音聊天、实时游戏同步）。

2. **支持WebRTC通信**  
   - UE的WebRTC功能（如像素流送、浏览器与UE间的实时交互）依赖DTLS加密数据传输。WebRTC强制要求使用DTLS-SRTP（安全实时传输协议）来保护音视频流。

3. **防止作弊与篡改**  
   - 在多人游戏中，DTLS可防止玩家篡改UDP数据包（例如位置坐标、伤害数值等），提升反作弊能力。

4. **身份验证**  
   - DTLS通过证书交换验证通信双方身份，确保客户端与服务器之间的可信连接（如防止伪造服务器）。

---

### UUE中DTLS的应用场景
- **像素流送（Pixel Streaming）**  
  UE通过WebRTC将画面流送到浏览器时，DTLS用于加密视频流和控制指令。
- **多人游戏网络**  
  如果游戏使用UDP协议（如Fast UDP或自定义方案），DTLS可加固通信安全。
- **物联网（IoT）与云交互**  
  设备与UE实例间的安全数据传输（如传感器数据）。

---

### 相关技术
- **DTLS vs TLS**  
  TLS用于TCP（如HTTPS），而DTLS是TLS的UDP适配版本，解决了UDP的丢包和乱序问题。
- **UE的实现**  
  UE通常通过第三方库（如OpenSSL、mbed TLS）或平台原生API（如WebRTC的DTLS）支持DTLS，开发者无需直接实现协议。

---

### 注意事项
- **性能开销**  
  DTLS会增加少量计算延迟（加密/解密），但对现代硬件影响较小。
- **配置复杂性**  
  需正确设置证书和密钥管理（如使用`SSL`模块或插件）。
- **版本依赖**  
  不同UE版本对DTLS的支持可能不同（尤其是WebRTC相关功能）。

如果需要具体实现细节，可以查阅UE的[WebRTC插件文档](https://docs.unrealengine.com/)或网络安全相关的API（如`USocketSubsystem`）。

## FEncryptionData 
FEncryptionData这个结构体是做什么用的.
``` cpp
struct FEncryptionData
{
	/** Encryption key */
	TArray<uint8> Key;
	/** Encryption fingerprint */
	TArray<uint8> Fingerprint;
	/** Encryption identifier */
	FString Identifier;
};
```

在 Unreal Engine (UE) 中，`FEncryptionData` 是一个用于存储加密相关数据的结构体，通常用于 **安全通信**（如 DTLS、TLS）或 **数据加密** 的场景。以下是该结构体成员的详细说明及其典型用途：

---

### 成员说明
1. **`Key`**  
   - 类型：`TArray<uint8>`（动态字节数组）  
   - 作用：存储**加密密钥**（如对称密钥、公钥/私钥的二进制形式）。  
   - 示例：  
     - DTLS/TLS 握手后生成的会话密钥。  
     - 用于加密游戏数据包的 AES-256 密钥。

2. **`Fingerprint`**  
   - 类型：`TArray<uint8>`  
   - 作用：存储密钥的**指纹**（如哈希值或证书指纹），用于验证密钥的合法性。  
   - 示例：  
     - WebRTC 中 DTLS 证书的 SHA-256 指纹，防止中间人攻击。  
     - 比较服务端和客户端密钥是否匹配。

3. **`Identifier`**  
   - 类型：`FString`  
   - 作用：标识加密数据的**唯一名称或标签**，便于调试或管理多组密钥。  
   - 示例：  
     - 为不同玩家分配唯一的加密标识（如 `Player1_Key`）。  
     - 区分不同会话的密钥（如 `SessionID_1234`）。

---

### 典型应用场景
1. **DTLS/TLS 安全通信**  
   - 在 UE 的 WebRTC 或网络模块中，`FEncryptionData` 可能用于存储 DTLS 握手阶段生成的密钥和证书指纹，确保 UDP 通信安全。

2. **数据包加密**  
   - 在多人游戏中，使用 `Key` 加密敏感数据（如玩家输入、位置坐标），并通过 `Fingerprint` 验证密钥有效性。

3. **像素流送（Pixel Streaming）**  
   - 加密浏览器与 UE 实例之间的视频流和控制指令时，`Identifier` 可能标识不同的流送会话。

4. **存档或资产加密**  
   - 将 `Key` 用于加密本地存档文件，`Fingerprint` 防止篡改。

---

### 示例代码
```cpp
// 创建并填充 FEncryptionData
FEncryptionData EncryptionData;
EncryptionData.Key = {0x01, 0x02, 0x03}; // 示例密钥
EncryptionData.Fingerprint = {0xFF, 0xEE}; // 示例指纹
EncryptionData.Identifier = TEXT("PlayerSession_123");

// 使用场景：传递给加密函数或网络模块
if (IsValidEncryption(EncryptionData.Fingerprint)) 
{
    EncryptPacket(EncryptionData.Key, RawData);
}
```

---

### 注意事项
1. **密钥管理**  
   - 避免硬编码 `Key`，应动态生成或从安全配置中加载。
2. **指纹算法**  
   - `Fingerprint` 通常通过哈希算法（如 SHA-256）生成，需与通信方协商一致。
3. **UE 内置支持**  
   - 结合 `USSL`（如 OpenSSL 插件）或 WebRTC 模块使用，无需手动实现加密逻辑。

如果需要进一步与 DTLS 集成，可以检查 UE 的 `IDTLSHandler` 接口或 WebRTC 相关源码（如 `PeerConnection`）。
## 总结
以上回答来自DeepSeek.