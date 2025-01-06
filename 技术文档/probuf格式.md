# Protobuf跨语言对象结构方案

## 一、消息结构定义

### 1.1 基础消息结构
```go
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

// 路由信息
message Route {
    CmdType cmd = 1;                  // 命令类型
    MsgType type = 2;                 // 消息类型
    string conversation = 3;          // 会话ID
    ConversationType conversation_type = 4;  // 会话类型(1单聊/2群聊/3聊天室等)
    string from = 5;                  // 发送者ID
    string to = 6;                    // 接收者ID
    repeated string target = 7;       // 目标用户列表
    string source = 8;                // 消息来源(serverApi/websocket)
}

// 消息内容
message Content {
    string text = 1;                  // 文本内容
    repeated ContentItem items = 2;    // 富文本内容
    Mentions mentions = 3;            // @功能
    Quote quote = 4;                  // 引用消息
    map<string, string> extension = 5; // 消息扩展字段
}

// 富文本内容项
message ContentItem {
    string type = 1;                  // 内容类型
    int32 start = 2;                 // 起始位置
    int32 end = 3;                   // 结束位置
    string uid = 4;                  // 用户ID(at类型时)
    string name = 5;                 // 显示名称
    string url = 6;                  // 资源URL
    string thumbnail = 7;            // 缩略图URL
    int32 width = 8;                // 宽度
    int32 height = 9;               // 高度
    int64 size = 10;                // 文件大小
    map<string, string> attrs = 11;  // 其他属性
}

// @功能
message Mentions {
    bool all = 1;                     // 是否@所有人
    repeated string uids = 2;         // @的用户列表
    string push_content = 3;          // @推送内容
}

// 引用消息
message Quote {
    string msg_id = 1;                // 引用消息ID
    string text = 2;                  // 引用内容
}

// 消息控制
message Control {
    bool persistent = 1;              // 是否持久化
    bool sync_device = 2;             // 是否多端同步
    int32 priority = 3;              // 消息优先级(0-9)
    int32 ttl = 4;                   // 消息有效期(秒)
    
    bool need_receipt = 5;            // 是否需要回执
    ReceiptType receipt_type = 6;     // 回执类型
    int32 read_timeout = 7;          // 已读超时时间(秒)
    
    PushConfig push = 8;             // 推送配置
    Security security = 9;           // 安全配置
    Callback callback = 10;          // 回调配置
}

// 推送配置
message PushConfig {
    bool enable = 1;                  // 是否开启推送
    string title = 2;                 // 推送标题
    string content = 3;               // 推送内容
    map<string, string> payload = 4;  // 推送自定义数据
    int32 badge = 5;                 // 角标数
    string sound = 6;                 // 推送声音
    string channel_id = 7;            // 推送通道
}

// 安全配置
message Security {
    bool check_sensitive = 1;         // 是否检查敏感词
    bool check_spam = 2;              // 是否检查垃圾消息
    bool encryption = 3;              // 是否加密
    string sign = 4;                  // 消息签名
}

// 回调配置
message Callback {
    bool enable = 1;                  // 是否开启回调
    string url = 2;                   // 回调地址
    repeated string events = 3;        // 需要回调的事件类型
    int32 timeout = 4;               // 回调超时时间(毫秒)
    RetryConfig retry = 5;           // 重试配置
    CallbackSecurity security = 6;    // 回调安全配置
}

// 重试配置
message RetryConfig {
    int32 max_times = 1;             // 最大重试次数
    int32 interval = 2;              // 重试间隔(毫秒)
}

// 回调安全配置
message CallbackSecurity {
    string token = 1;                 // 回调验证token
    string sign_type = 2;             // 签名类型
}

// 业务选项
message Options {
    string biz_type = 1;              // 业务类型
    string biz_id = 2;                // 业务ID
    Conversation conversation = 3;     // 会话配置
    map<string, string> custom = 4;   // 自定义业务字段
    map<string, string> extra = 5;    // 额外扩展字段
}

// 会话配置
message Conversation {
    bool top = 1;                     // 是否置顶会话
    bool mute = 2;                    // 是否免打扰
    bool unread = 3;                  // 是否计入未读
}

// 响应消息
message IMResponse {
    Header header = 1;                // 消息头
    Status status = 2;                // 状态信息
    ResponseData data = 3;            // 响应数据
}

// 状态信息
message Status {
    int32 code = 1;                   // 状态码
    string message = 2;               // 状态描述
    map<string, string> detail = 3;   // 详细错误信息
}

// 响应数据
message ResponseData {
    string msg_id = 1;                // 服务端消息ID
    int64 room_seq = 2;              // 会话序列号
    int64 user_pts = 3;              // 用户序列号
    int64 timestamp = 4;              // 服务器时间戳
    Results results = 5;              // 消息处理结果
}

// 处理结果
message Results {
    repeated string sensitive_words = 1;  // 命中的敏感词
    string replaced_text = 2;            // 替换后的文本
    float spam_score = 3;               // 垃圾消息分数
    repeated string push_results = 4;    // 推送结果
}

// WebRTC信令消息
message WebRTCSignal {
    string call_id = 1;               // 通话ID
    int32 call_type = 2;             // 1:音频 2:视频
    int32 media_type = 3;            // 1:音频 2:视频 3:屏幕共享
    string sdp = 4;                  // SDP信息
    repeated RTCCandidate candidates = 5; // ICE候选者
}

// ICE候选者
message RTCCandidate {
    string sdp_mid = 1;              // SDP中线路ID
    int32 sdp_m_line_index = 2;      // SDP中线路索引
    string candidate = 3;            // 候选者信息
}

// 通话状态
message CallStatus {
    string call_id = 1;              // 通话ID
    int32 status = 2;               // 1:已接通 2:结束通话 3:通话中断
    int32 duration = 3;             // 通话时长(秒)
    string reason = 4;              // 结束原因
}

// 枚举定义
enum CmdType {
    // === 消息命令 ===
    SINGLE_CHAT = 0;                 // 单聊消息
    GROUP_CHAT = 1;                  // 群聊消息
    CHATROOM_CHAT = 2;              // 聊天室消息
    BROADCAST = 3;                   // 广播消息
    CUSTOM_CHAT = 4;                 // 自定义消息
    
    // === 消息操作 ===
    MSG_RECALL = 11;                // 撤回消息
    MSG_DELETE = 12;                // 删除消息
    MSG_EDIT = 13;                  // 编辑消息
    MSG_REACTION = 14;              // 消息回应
    
    // === 已读命令 ===
    MSG_RECEIPT = 21;               // 消息回执
    MSG_READ = 22;                  // 消息已读
    READ_ALL = 23;                  // 全部已读
    READ_CANCEL = 24;               // 取消已读
    
    // === 会话命令 ===
    CONVERSATION_CREATE = 31;        // 创建会话
    CONVERSATION_DELETE = 32;        // 删除会话
    CONVERSATION_UPDATE = 33;        // 更新会话
    CONVERSATION_SYNC = 34;         // 同步会话
    
    // === 群组命令 ===
    GROUP_CREATE = 41;              // 创建群组
    GROUP_JOIN = 42;                // 加入群组
    GROUP_LEAVE = 43;               // 退出群组
    GROUP_DISMISS = 44;             // 解散群组
    GROUP_UPDATE = 45;              // 更新群组
    GROUP_MUTE = 46;                // 群组禁言
    
    // === 用户命令 ===
    USER_MUTE = 51;                 // 用户禁言
    USER_BLOCK = 52;                // 用户拉黑
    USER_ONLINE = 53;               // 用户上线
    USER_OFFLINE = 54;              // 用户离线
    USER_SETTING = 55;              // 用户设置
    
    // === 关系命令 ===
    FRIEND_ADD = 61;                // 添加好友
    FRIEND_DELETE = 62;             // 删除好友
    FRIEND_UPDATE = 63;             // 更新好友
    FRIEND_BLOCK = 64;              // 拉黑好友
    
    // === 同步命令 ===
    SYNC_MSG = 71;                  // 同步消息
    SYNC_CONVERSATION = 72;         // 同步会话
    SYNC_GROUP = 73;                // 同步群组
    SYNC_FRIEND = 74;               // 同步好友
    
    // === 系统命令 ===
    SYS_NOTIFY = 91;                // 系统通知
    CUSTOM_NOTIFY = 92;             // 自定义通知
    ERROR = 99;                     // 错误消息
    
    // === 输入状态命令 ===
    TYPING_STATUS = 81;             // 输入状态
    TYPING_CANCEL = 82;             // 取消输入
    
    // === WebRTC信令命令 ===
    WEBRTC_INVITE = 101;            // 音视频邀请
    WEBRTC_ANSWER = 102;            // 音视频应答
    WEBRTC_CANDIDATE = 103;         // 候选者更新
    WEBRTC_STATUS = 104;            // 通话状态
    WEBRTC_SWITCH = 105;            // 切换音视频
    WEBRTC_QUALITY = 106;           // 质量报告
    
    // === 屏幕共享命令 ===
    SCREEN_SHARE_START = 111;       // 开始共享
    SCREEN_SHARE_STOP = 112;        // 停止共享
    SCREEN_SHARE_QUALITY = 113;     // 共享质量
}

enum MsgType {
    // === 基础消息 ===
    TEXT = 0;                       // 文本消息
    IMAGE = 1;                      // 图片消息
    AUDIO = 2;                      // 语音消息
    VIDEO = 3;                      // 视频消息
    FILE = 4;                       // 文件消息
    LOCATION = 5;                   // 位置消息
    CONTACT = 6;                    // 名片消息
    
    // === 富媒体消息 ===
    RICH_TEXT = 11;                 // 富文本消息
    MARKDOWN = 12;                  // Markdown消息
    HTML = 13;                      // HTML消息
    
    // === 复合消息 ===
    QUOTE = 21;                     // 引用消息
    FORWARD = 22;                   // 转发消息
    MERGE = 23;                     // 合并消息
    CARD = 24;                      // 卡片消息
    
    // === 互动消息 ===
    REACTION = 31;                  // 表情回应
    VOTE = 32;                      // 投票消息
    RED_PACKET = 33;                // 红包消息
    GAME = 34;                      // 小游戏消息
    
    // === 通知消息 ===
    GROUP_NOTIFY = 41;              // 群通知
    FRIEND_NOTIFY = 42;             // 好友通知
    SYSTEM_NOTIFY = 43;             // 系统通知
    CUSTOM_NOTIFY = 44;             // 自定义通知
    
    // === 状态消息 ===
    TYPING = 51;                    // 正在输入
    TYPING_END = 52;                // 结束输入
    
    // === WebRTC信令类型 ===
    WEBRTC_SIGNAL = 61;             // 音视频信令
    WEBRTC_STATUS = 62;             // 通话状态
    WEBRTC_QUALITY = 63;            // 质量数据
    
    // === 屏幕共享类型 ===
    SCREEN_SHARE = 71;              // 屏幕共享
    SCREEN_CONTROL = 72;            // 共享控制
    
    // === 自定义消息 ===
    CUSTOM = 91;                    // 自定义消息
    TEMPLATE = 92;                  // 模板消息
}

enum ConversationType {
    SINGLE = 1;                     // 单聊
    GROUP = 2;                      // 群聊
    CHATROOM = 3;                   // 聊天室
    //机器人
    BOT = 4;
    // 频道
    CHANNEL = 5;
}

enum ReceiptType {
    DELIVERED = 0;                  // 送达回执
    READ = 1;                       // 已读回执
}
```

## 二、各平台使用示例

### 2.1 Golang使用
```go
// core/message.go
package imsdk

import (
    "time"
    "github.com/your/project/proto"
)

func SendTextMessage(conversationId, fromId, toId string, text string) error {
    msg := &proto.IMMessage{
        Header: &proto.Header{
            Ver: 1,
            Seq: generateSeq(),
            AppId: "app_123",
            Platform: "golang",
            Timestamp: time.Now().UnixMilli(),
        },
        Route: &proto.Route{
            Cmd: proto.CmdType_SINGLE_CHAT,
            Type: proto.MsgType_TEXT,
            Conversation: conversationId,
            ConversationType: proto.ConversationType_SINGLE,
            From: fromId,
            To: toId,
        },
        Content: &proto.Content{
            Text: text,
        },
        Control: &proto.Control{
            Persistent: true,
            SyncDevice: true,
            Push: &proto.PushConfig{
                Enable: true,
                Content: text,
            },
        },
    }
    
    data, err := proto.Marshal(msg)
    if err != nil {
        return err
    }
    
    return client.Send(data)
}
```

### 2.2 Swift使用
```swift
// iOS/IMClient.swift
import IMProto

class IMClient {
    func sendTextMessage(conversationId: String, fromId: String, toId: String, text: String) {
        let header = IMHeader.with {
            $0.ver = 1
            $0.seq = generateSeq()
            $0.appID = "app_123"
            $0.platform = "ios"
            $0.timestamp = Int64(Date().timeIntervalSince1970 * 1000)
        }
        
        let route = IMRoute.with {
            $0.cmd = .singleChat
            $0.type = .text
            $0.conversation = conversationId
            $0.conversationType = .single
            $0.from = fromId
            $0.to = toId
        }
        
        let content = IMContent.with {
            $0.text = text
        }
        
        let control = IMControl.with {
            $0.persistent = true
            $0.syncDevice = true
            $0.push = IMPushConfig.with {
                $0.enable = true
                $0.content = text
            }
        }
        
        let message = IMMessage.with {
            $0.header = header
            $0.route = route
            $0.content = content
            $0.control = control
        }
        
        try? client.send(message.serializedData())
    }
}
```

### 2.3 Kotlin使用
```kotlin
// Android/IMClient.kt
import com.your.project.proto.*

class IMClient {
    fun sendTextMessage(conversationId: String, fromId: String, toId: String, text: String) {
        val message = IMMessage.newBuilder()
            .setHeader(Header.newBuilder()
                .setVer(1)
                .setSeq(generateSeq())
                .setAppId("app_123")
                .setPlatform("android")
                .setTimestamp(System.currentTimeMillis())
                .build())
            .setRoute(Route.newBuilder()
                .setCmd(CmdType.SINGLE_CHAT)
                .setType(MsgType.TEXT)
                .setConversation(conversationId)
                .setConversationType(ConversationType.SINGLE)
                .setFrom(fromId)
                .setTo(toId)
                .build())
            .setContent(Content.newBuilder()
                .setText(text)
                .build())
            .setControl(Control.newBuilder()
                .setPersistent(true)
                .setSyncDevice(true)
                .setPush(PushConfig.newBuilder()
                    .setEnable(true)
                    .setContent(text)
                    .build())
                .build())
            .build()
            
        client.send(message.toByteArray())
    }
}
```

### 2.4 TypeScript使用
```typescript
// Web/IMClient.ts
import { IMMessage, Header, Route, Content, Control, PushConfig, CmdType, MsgType, ConversationType } from './proto/im';

class IMClient {
    sendTextMessage(conversationId: string, fromId: string, toId: string, text: string) {
        const message = IMMessage.create({
            header: {
                ver: 1,
                seq: this.generateSeq(),
                appId: "app_123",
                platform: "web",
                timestamp: Date.now()
            },
            route: {
                cmd: CmdType.SINGLE_CHAT,
                type: MsgType.TEXT,
                conversation: conversationId,
                conversationType: ConversationType.SINGLE,
                from: fromId,
                to: toId
            },
            content: {
                text: text
            },
            control: {
                persistent: true,
                syncDevice: true,
                push: {
                    enable: true,
                    content: text
                }
            }
        });
        
        this.send(IMMessage.encode(message).finish());
    }
}
```

## 三、注意事项

### 3.1 版本管理
1. 使用proto3语法
2. 保持字段向后兼容
3. 不要删除或修改已有字段
4. 新增字段使用递增编号

### 3.2 编译部署
1. 统一使用protoc编译
2. 保持各端proto文件同步
3. 添加编译脚本到CI/CD
4. 自动生成文档说明

### 3.3 性能优化
1. 使用对象池复用Message
2. 避免频繁创建临时对象
3. 批量处理消息
4. 压缩传输数据

### 3.4 安全考虑
1. 加密传输数据
2. 验证消息完整性
3. 检查数据合法性
4. 控制消息大小