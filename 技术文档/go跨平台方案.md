# Golang跨平台IM SDK实现方案

## 目录
- [1. 整体架构](#1-整体架构)
- [2. Protocol Buffers统一管理](#2-protocol-buffers统一管理)
- [3. 核心实现](#3-核心实现)
- [4. 跨平台支持](#4-跨平台支持)
- [5. 构建部署](#5-构建部署)
- [6. 性能优化](#6-性能优化)
- [7. 最佳实践](#7-最佳实践)
- [8. 各端使用示例](#8-各端使用示例)

## 1. 整体架构

### 1.1 架构图
```
+------------------+     +------------------+     +------------------+
|     iOS SDK      |     |   Android SDK    |     |    Web SDK      |
|  SwiftProtobuf   |     |  Kotlin Proto    |     |   protobuf.js   |
+------------------+     +------------------+     +------------------+
          |                      |                        |
          v                      v                        v
+----------------------------------------------------------+
|                     Golang Core SDK                        |
|                  Protocol Buffers Core                     |
+----------------------------------------------------------+
|  WebSocket  |  Message Queue  |  State Machine | Protocol  |
+----------------------------------------------------------+
          |                      |                        |
          v                      v                        v
+----------------------------------------------------------+
|                      Backend Server                        |
|                    Protobuf Messages                      |
+----------------------------------------------------------+
```



整体架构

```
+------------------+    +------------------+    +------------------+
|   平台层          |    |    核心层        |    |    协议层        |
|  Platform Layer  |    |   Core Layer    |    | Protocol Layer  |
+------------------+    +------------------+    +------------------+
| - Android SDK    |    | - 连接管理        |    | - 消息编解码     |
| - iOS SDK        |    | - 消息收发        |    | - 协议定义      |
| - Web SDK        |    | - 状态同步        |    | - 序列化       |
| - PC SDK         |    | - 会话管理        |    | - 压缩算法      |
+------------------+    +------------------+    +------------------+
```

### 1.2 核心模块
- Protocol Buffers消息定义
- WebSocket连接管理
- 消息收发处理
- 状态同步管理
- 会话管理
- 存储管理
- 网络状态管理

### 1.3 技术选型
- 核心语言: Golang 1.21+
- 消息序列化: Protocol Buffers 3
- WebSocket库: gorilla/websocket
- 跨平台桥接: CGO
- 移动端打包: gomobile
- Web支持: WebAssembly

## 2. Protocol Buffers统一管理

### 2.1 Proto文件结构
```proto
// proto/im.proto
syntax = "proto3";

package im;

option go_package = "github.com/your/project/proto";
option java_package = "com.your.project.proto";
option swift_prefix = "IM";

// 完整消息结构
message IMMessage {
    Header header = 1;
    Route route = 2;
    Content content = 3;
    Control control = 4;
    Options options = 5;
}

// 消息头
message Header {
    int32 ver = 1;                    // 协议版本号
    string seq = 2;                   // 客户端序列号
    string app_id = 3;                // 应用ID
    string tenant_id = 4;             // 租户ID
    string platform = 5;              // 平台类型
    string device_id = 6;             // 设备ID
    int64 timestamp = 7;              // 客户端发送时间
}

// 更多消息定义...
```

### 2.2 Proto编译配置
```yaml
# proto.yaml
protoc:
  version: 3.21.0
  plugins:
    - name: go
      version: 1.28.0
      options: paths=source_relative
    - name: swift
      version: 1.19.0
    - name: kotlin
      version: 1.6.0
    - name: ts
      version: 3.20.1
      options: import_style=commonjs,binary

targets:
  - lang: go
    out: server/proto
  - lang: swift
    out: ios/IMClient/Proto
  - lang: kotlin
    out: android/imclient/src/main/java
  - lang: ts
    out: web/src/proto
```

### 2.3 消息版本管理
```go
// 版本兼容策略
const (
    MinSupportVersion = 1
    CurrentVersion = 2
)

// 版本检查
func checkVersion(msg *proto.IMMessage) error {
    if msg.Header.Ver < MinSupportVersion {
        return ErrVersionTooOld
    }
    if msg.Header.Ver > CurrentVersion {
        return ErrVersionTooNew
    }
    return nil
}
```

## 3. 核心实现

### 3.1 基础结构定义
```go
// core/client.go
package imsdk

import (
    "github.com/your/project/proto"
)

type IMClient interface {
    Connect() error
    Disconnect() error
    SendMessage(*proto.IMMessage) error
    OnMessage(func(*proto.IMMessage))
    GetState() State
}

// 客户端实现
type wsClient struct {
    conn      *websocket.Conn
    options   *Options
    handlers  map[string]func(*proto.IMMessage)
    msgChan   chan *proto.IMMessage
    closeChan chan struct{}
    state     atomic.Value
    mu        sync.RWMutex
}
```

### 3.2 消息处理实现
```go
// core/message.go
package imsdk

func (c *wsClient) handleMessage(data []byte) error {
    // 反序列化消息
    msg := acquireMessage()
    if err := proto.Unmarshal(data, msg); err != nil {
        releaseMessage(msg)
        return err
    }

    // 版本检查
    if err := checkVersion(msg); err != nil {
        releaseMessage(msg)
        return err
    }

    // 处理消息
    if handler, ok := c.handlers[msg.Route.Cmd.String()]; ok {
        handler(msg)
    }

    releaseMessage(msg)
    return nil
}

func (c *wsClient) SendMessage(msg *proto.IMMessage) error {
    // 设置基础字段
    msg.Header = &proto.Header{
        Ver: CurrentVersion,
        Seq: generateSeq(),
        Timestamp: time.Now().UnixMilli(),
    }

    // 序列化消息
    data, err := proto.Marshal(msg)
    if err != nil {
        return err
    }

    return c.conn.WriteMessage(websocket.BinaryMessage, data)
}
```

## 4. 跨平台支持

### 4.1 iOS SDK实现
```swift
// iOS/IMClient.swift
import IMProto

class IMClient {
    func sendMessage(_ text: String) {
        let msg = IMMessage.with {
            $0.header = IMHeader.with {
                $0.ver = 2
                $0.seq = generateSeq()
                $0.timestamp = Int64(Date().timeIntervalSince1970 * 1000)
            }
            $0.content = IMContent.with {
                $0.text = text
            }
        }
        
        // 序列化并发送
        guard let data = try? msg.serializedData() else { return }
        webSocket.send(data)
    }
}
```

### 4.2 Android SDK实现
```kotlin
// Android/IMClient.kt
import com.your.project.proto.*

class IMClient {
    fun sendMessage(text: String) {
        val msg = IMMessage.newBuilder()
            .setHeader(IMHeader.newBuilder()
                .setVer(2)
                .setSeq(generateSeq())
                .setTimestamp(System.currentTimeMillis())
                .build())
            .setContent(IMContent.newBuilder()
                .setText(text)
                .build())
            .build()
            
        webSocket.send(msg.toByteArray())
    }
}
```

### 4.3 Web SDK实现
```typescript
// Web/IMClient.ts
import * as protobuf from 'protobufjs'
import { IMMessage, IMHeader, IMContent } from './proto/im'

class IMClient {
    async sendMessage(text: string) {
        const msg = IMMessage.create({
            header: {
                ver: 2,
                seq: this.generateSeq(),
                timestamp: Date.now()
            },
            content: {
                text: text
            }
        })
        
        const buffer = IMMessage.encode(msg).finish()
        this.webSocket.send(buffer)
    }
}
```

## 5. 构建部署

### 5.1 Proto编译脚本
```bash
#!/bin/bash

# 编译proto文件
function compile_proto() {
    # Go
    protoc --go_out=./server/proto --go_opt=paths=source_relative proto/*.proto
    
    # Swift
    protoc --swift_out=./ios/IMClient/Proto proto/*.proto
    
    # Kotlin
    protoc --kotlin_out=./android/imclient/src/main/java proto/*.proto
    
    # TypeScript
    protoc --ts_out=./web/src/proto proto/*.proto
}

# 清理生成的文件
function clean_proto() {
    rm -rf ./server/proto/*
    rm -rf ./ios/IMClient/Proto/*
    rm -rf ./android/imclient/src/main/java/com/your/project/proto/*
    rm -rf ./web/src/proto/*
}

# 执行编译
clean_proto
compile_proto
```

### 5.2 依赖配置

iOS (Podfile):
```ruby
pod 'SwiftProtobuf'
```

Android (build.gradle):
```gradle
dependencies {
    implementation 'com.google.protobuf:protobuf-java:3.21.0'
}
```

Web (package.json):
```json
{
  "dependencies": {
    "protobufjs": "^7.0.0"
  }
}
```

## 6. 性能优化

### 6.1 消息对象池
```go
// core/pool.go
var messagePool = sync.Pool{
    New: func() interface{} {
        return &proto.IMMessage{}
    },
}

func acquireMessage() *proto.IMMessage {
    return messagePool.Get().(*proto.IMMessage)
}

func releaseMessage(msg *proto.IMMessage) {
    msg.Reset()
    messagePool.Put(msg)
}
```

### 6.2 批量消息处理
```go
type MessageBatch struct {
    messages []*proto.IMMessage
    size     int
}

func (b *MessageBatch) Add(msg *proto.IMMessage) {
    if b.size >= len(b.messages) {
        b.flush()
    }
    b.messages[b.size] = msg
    b.size++
}
```

## 7. 最佳实践

### 7.1 Proto文件管理
1. 版本控制
   - 使用语义化版本
   - 保持向后兼容
   - 谨慎删除字段

2. 字段命名规范
   - 使用下划线命名法
   - 字段名要有意义
   - 添加详细注释

3. 消息结构设计
   - 合理划分消息类型
   - 避免过深的嵌套
   - 考虑扩展性

### 7.2 序列化优化
1. 消息大小控制
   - 避免过大的消息
   - 使用压缩算法
   - 分片传输大消息

2. 性能优化
   - 使用对象池
   - 避免频繁创建对象
   - 批量处理消息

3. 安全考虑
   - 加密敏感字段
   - 校验消息完整性
   - 防止重放攻击

## 8. 各端使用示例

### 8.1 iOS端使用示例
```swift
// 初始化SDK配置
let config = IMConfig(
    appId: "your_app_id",
    userId: "user123",
    token: "your_token",
    serverUrl: "wss://im.example.com/ws"
)

// 创建SDK实例
let imClient = IMClient.shared

// 初始化并连接
imClient.initialize(config: config) { error in
    if let error = error {
        print("初始化失败: \(error)")
        return
    }
    print("初始化成功")
}

// 注册消息监听
imClient.onMessage { message in
    // 处理收到的消息
    print("收到消息: \(message.content.text)")
}

// 发送文本消息
let message = IMMessage.with {
    $0.header = IMHeader.with {
        $0.ver = 2
        $0.seq = UUID().uuidString
        $0.timestamp = Int64(Date().timeIntervalSince1970 * 1000)
        $0.platform = "ios"
    }
    $0.route = IMRoute.with {
        $0.cmd = .chat
        $0.msgType = .text
        $0.conversationType = .single
        $0.senderId = "user123"
        $0.receiverId = "user456"
    }
    $0.content = IMContent.with {
        $0.text = "你好，这是一条测试消息"
    }
}

imClient.sendMessage(message) { error in
    if let error = error {
        print("发送失败: \(error)")
        return
    }
    print("发送成功")
}

// 发送图片消息
let imageMessage = IMMessage.with {
    $0.header = IMHeader.with {
        $0.ver = 2
        $0.seq = UUID().uuidString
        $0.timestamp = Int64(Date().timeIntervalSince1970 * 1000)
        $0.platform = "ios"
    }
    $0.route = IMRoute.with {
        $0.cmd = .chat
        $0.msgType = .image
        $0.conversationType = .single
        $0.senderId = "user123"
        $0.receiverId = "user456"
    }
    $0.content = IMContent.with {
        $0.mediaContent = IMMediaContent.with {
            $0.url = "https://example.com/image.jpg"
            $0.size = 1024 * 1024
            $0.width = 800
            $0.height = 600
            $0.format = "jpg"
        }
    }
}

imClient.sendMessage(imageMessage) { error in
    if let error = error {
        print("发送图片失败: \(error)")
        return
    }
    print("发送图片成功")
}
```

### 8.2 Android端使用示例
```kotlin
// 初始化SDK配置
val config = IMConfig.Builder()
    .setAppId("your_app_id")
    .setUserId("user123")
    .setToken("your_token")
    .setServerUrl("wss://im.example.com/ws")
    .build()

// 创建SDK实例
val imClient = IMClient.getInstance()

// 初始化并连接
imClient.initialize(config, object : IMCallback<Unit> {
    override fun onSuccess(result: Unit) {
        Log.d(TAG, "初始化成功")
    }

    override fun onError(error: IMError) {
        Log.e(TAG, "初始化失败: ${error.message}")
    }
})

// 注册消息监听
imClient.setMessageListener { message ->
    // 处理收到的消息
    Log.d(TAG, "收到消息: ${message.content.text}")
}

// 发送文本消息
val message = IMMessage.newBuilder()
    .setHeader(IMHeader.newBuilder()
        .setVer(2)
        .setSeq(UUID.randomUUID().toString())
        .setTimestamp(System.currentTimeMillis())
        .setPlatform("android")
        .build())
    .setRoute(IMRoute.newBuilder()
        .setCmd(CommandType.CHAT)
        .setMsgType(MessageType.TEXT)
        .setConversationType(ConversationType.SINGLE)
        .setSenderId("user123")
        .setReceiverId("user456")
        .build())
    .setContent(IMContent.newBuilder()
        .setText("你好，这是一条测试消息")
        .build())
    .build()

imClient.sendMessage(message, object : IMCallback<Unit> {
    override fun onSuccess(result: Unit) {
        Log.d(TAG, "发送成功")
    }

    override fun onError(error: IMError) {
        Log.e(TAG, "发送失败: ${error.message}")
    }
})

// 发送图片消息
val imageMessage = IMMessage.newBuilder()
    .setHeader(IMHeader.newBuilder()
        .setVer(2)
        .setSeq(UUID.randomUUID().toString())
        .setTimestamp(System.currentTimeMillis())
        .setPlatform("android")
        .build())
    .setRoute(IMRoute.newBuilder()
        .setCmd(CommandType.CHAT)
        .setMsgType(MessageType.IMAGE)
        .setConversationType(ConversationType.SINGLE)
        .setSenderId("user123")
        .setReceiverId("user456")
        .build())
    .setContent(IMContent.newBuilder()
        .setMediaContent(IMMediaContent.newBuilder()
            .setUrl("https://example.com/image.jpg")
            .setSize(1024 * 1024)
            .setWidth(800)
            .setHeight(600)
            .setFormat("jpg")
            .build())
        .build())
    .build()

imClient.sendMessage(imageMessage, object : IMCallback<Unit> {
    override fun onSuccess(result: Unit) {
        Log.d(TAG, "发送图片成功")
    }

    override fun onError(error: IMError) {
        Log.e(TAG, "发送图片失败: ${error.message}")
    }
})
```

### 8.3 Web端使用示例
```typescript
// 初始化SDK配置
const config: IMConfig = {
    appId: 'your_app_id',
    userId: 'user123',
    token: 'your_token',
    serverUrl: 'wss://im.example.com/ws'
};

// 创建SDK实例
const imClient = new IMClient();

// 初始化并连接
await imClient.initialize(config);
console.log('初始化成功');

// 注册消息监听
imClient.onMessage((message: IMMessage) => {
    // 处理收到的消息
    console.log('收到消息:', message.content.text);
});

// 发送文本消息
const message = IMMessage.create({
    header: {
        ver: 2,
        seq: generateUUID(),
        timestamp: Date.now(),
        platform: 'web'
    },
    route: {
        cmd: CommandType.CHAT,
        msgType: MessageType.TEXT,
        conversationType: ConversationType.SINGLE,
        senderId: 'user123',
        receiverId: 'user456'
    },
    content: {
        text: '你好，这是一条测试消息'
    }
});

try {
    await imClient.sendMessage(message);
    console.log('发送成功');
} catch (error) {
    console.error('发送失败:', error);
}

// 发送图片消息
const imageMessage = IMMessage.create({
    header: {
        ver: 2,
        seq: generateUUID(),
        timestamp: Date.now(),
        platform: 'web'
    },
    route: {
        cmd: CommandType.CHAT,
        msgType: MessageType.IMAGE,
        conversationType: ConversationType.SINGLE,
        senderId: 'user123',
        receiverId: 'user456'
    },
    content: {
        mediaContent: {
            url: 'https://example.com/image.jpg',
            size: 1024 * 1024,
            width: 800,
            height: 600,
            format: 'jpg'
        }
    }
});

try {
    await imClient.sendMessage(imageMessage);
    console.log('发送图片成功');
} catch (error) {
    console.error('发送图片失败:', error);
}
```

### 8.4 Flutter端使用示例
```dart
// 初始化SDK配置
final config = IMConfig(
    appId: 'your_app_id',
    userId: 'user123',
    token: 'your_token',
    serverUrl: 'wss://im.example.com/ws'
);

// 创建SDK实例
final imClient = IMClient();

// 初始化并连接
try {
    await imClient.initialize(config);
    print('初始化成功');
} catch (error) {
    print('初始化失败: $error');
}

// 注册消息监听
imClient.onMessage((IMMessage message) {
    // 处理收到的消息
    print('收到消息: ${message.content.text}');
});

// 发送文本消息
final message = IMMessage(
    header: IMHeader(
        ver: 2,
        seq: generateUUID(),
        timestamp: DateTime.now().millisecondsSinceEpoch,
        platform: 'flutter'
    ),
    route: IMRoute(
        cmd: CommandType.chat,
        msgType: MessageType.text,
        conversationType: ConversationType.single,
        senderId: 'user123',
        receiverId: 'user456'
    ),
    content: IMContent(
        text: '你好，这是一条测试消息'
    )
);

try {
    await imClient.sendMessage(message);
    print('发送成功');
} catch (error) {
    print('发送失败: $error');
}

// 发送图片消息
final imageMessage = IMMessage(
    header: IMHeader(
        ver: 2,
        seq: generateUUID(),
        timestamp: DateTime.now().millisecondsSinceEpoch,
        platform: 'flutter'
    ),
    route: IMRoute(
        cmd: CommandType.chat,
        msgType: MessageType.image,
        conversationType: ConversationType.single,
        senderId: 'user123',
        receiverId: 'user456'
    ),
    content: IMContent(
        mediaContent: IMMediaContent(
            url: 'https://example.com/image.jpg',
            size: 1024 * 1024,
            width: 800,
            height: 600,
            format: 'jpg'
        )
    )
);

try {
    await imClient.sendMessage(imageMessage);
    print('发送图片成功');
} catch (error) {
    print('发送图片失败: $error');
}
```

### 8.5 Electron端使用示例
```typescript
// 初始化SDK配置
const config: IMConfig = {
    appId: 'your_app_id',
    userId: 'user123',
    token: 'your_token',
    serverUrl: 'wss://im.example.com/ws'
};

// 创建SDK实例
const imClient = new IMClient();

// 初始化并连接
try {
    await imClient.initialize(config);
    console.log('初始化成功');
} catch (error) {
    console.error('初始化失败:', error);
}

// 注册消息监听
imClient.onMessage((message: IMMessage) => {
    // 处理收到的消息
    console.log('收到消息:', message.content.text);
    
    // 发送系统通知
    new Notification('新消息', {
        body: message.content.text
    });
});

// 发送文本消息
const message = IMMessage.create({
    header: {
        ver: 2,
        seq: generateUUID(),
        timestamp: Date.now(),
        platform: 'electron'
    },
    route: {
        cmd: CommandType.CHAT,
        msgType: MessageType.TEXT,
        conversationType: ConversationType.SINGLE,
        senderId: 'user123',
        receiverId: 'user456'
    },
    content: {
        text: '你好，这是一条测试消息'
    }
});

try {
    await imClient.sendMessage(message);
    console.log('发送成功');
} catch (error) {
    console.error('发送失败:', error);
}

// 发送文件消息
async function sendFile(filePath: string) {
    const fileStats = await fs.promises.stat(filePath);
    const fileExt = path.extname(filePath).slice(1);
    
    const fileMessage = IMMessage.create({
        header: {
            ver: 2,
            seq: generateUUID(),
            timestamp: Date.now(),
            platform: 'electron'
        },
        route: {
            cmd: CommandType.CHAT,
            msgType: MessageType.FILE,
            conversationType: ConversationType.SINGLE,
            senderId: 'user123',
            receiverId: 'user456'
        },
        content: {
            mediaContent: {
                localPath: filePath,
                size: fileStats.size,
                format: fileExt
            }
        }
    });

    try {
        await imClient.sendMessage(fileMessage);
        console.log('发送文件成功');
    } catch (error) {
        console.error('发送文件失败:', error);
    }
}
```

### 8.6 命令行工具示例
```go
// cmd/imcli/main.go
package main

import (
    "flag"
    "fmt"
    "os"
    "time"

    "github.com/your/project/imsdk"
    "github.com/your/project/proto"
)

var (
    appID     = flag.String("app", "", "应用ID")
    userID    = flag.String("user", "", "用户ID")
    token     = flag.String("token", "", "用户Token")
    serverURL = flag.String("server", "", "服务器地址")
)

func main() {
    flag.Parse()

    // 检查必要参数
    if *appID == "" || *userID == "" || *token == "" || *serverURL == "" {
        flag.Usage()
        os.Exit(1)
    }

    // 创建SDK实例
    client := imsdk.NewClient(&imsdk.Options{
        AppID:     *appID,
        UserID:    *userID,
        Token:     *token,
        ServerURL: *serverURL,
    })

    // 注册消息处理
    client.OnMessage(func(msg *proto.IMMessage) {
        fmt.Printf("收到消息: %s\n", msg.Content.Text)
    })

    // 连接服务器
    if err := client.Connect(); err != nil {
        fmt.Printf("连接失败: %v\n", err)
        os.Exit(1)
    }
    defer client.Disconnect()

    fmt.Println("连接成功")

    // 发送测试消息
    msg := &proto.IMMessage{
        Header: &proto.Header{
            Ver:       2,
            Seq:      fmt.Sprintf("%d", time.Now().UnixNano()),
            Timestamp: time.Now().UnixMilli(),
            Platform:  "cli",
        },
        Route: &proto.Route{
            Cmd:             proto.CommandType_CHAT,
            MsgType:         proto.MessageType_TEXT,
            ConversationType: proto.ConversationType_SINGLE,
            SenderId:        *userID,
            ReceiverId:      "user456",
        },
        Content: &proto.Content{
            Text: "你好，这是一条命令行发送的测试消息",
        },
    }

    if err := client.SendMessage(msg); err != nil {
        fmt.Printf("发送失败: %v\n", err)
        os.Exit(1)
    }

    fmt.Println("发送成功")
    time.Sleep(time.Second) // 等待消息发送完成
}
```

这些示例代码展示了:

1. 各端SDK的初始化配置
2. 连接服务器
3. 注册消息监听
4. 发送不同类型的消息(文本、图片、文件等)
5. 错误处理
6. 命令行工具的使用方式

每个平台都保持了相同的消息结构和处理流程，只是在具体实现上根据平台特性做了适配。所有平台都使用Protocol Buffers进行消息序列化，确保了跨平台的消息一致性。