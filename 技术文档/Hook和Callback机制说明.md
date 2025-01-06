# Hook和Callback机制说明

## 一、基本概念

### 1.1 Hook（钩子）
Hook是一种拦截机制，在程序执行流程的特定点插入自定义代码，可以影响或改变原有的执行流程。

特点：
- 同步执行
- 可以改变程序流程
- 作用范围局部
- 系统主动调用

### 1.2 Callback（回调）
Callback是一种异步通知机制，用于在某个操作完成后通知调用方。

特点：
- 异步执行
- 不改变程序流程
- 作用范围广泛
- 被动等待通知

## 二、在IM系统中的应用

### 2.1 Hook应用场景

#### 2.1.1 消息发送前Hook
```java
public interface MessageSendHook {
    // 返回true允许发送，false拦截发送
    boolean beforeMessageSend(Message message);
    
    // 可以修改消息内容
    Message processMessage(Message message);
}

// 实现示例
public class SensitiveWordHook implements MessageSendHook {
    @Override
    public boolean beforeMessageSend(Message message) {
        // 敏感词检查
        return !containsSensitiveWords(message.getContent());
    }
    
    @Override
    public Message processMessage(Message message) {
        // 替换敏感词
        String filtered = filterSensitiveWords(message.getContent());
        message.setContent(filtered);
        return message;
    }
}
```

#### 2.1.2 用户上线Hook
```java
public interface UserOnlineHook {
    // 用户上线前的处理
    boolean beforeUserOnline(String userId, String deviceId);
    
    // 可以修改用户状态
    UserStatus processUserStatus(UserStatus status);
}
```

#### 2.1.3 群消息Hook
```java
public interface GroupMessageHook {
    // 群消息发送前权限检查
    boolean beforeGroupMessage(String groupId, String userId, Message message);
    
    // 群消息内容处理
    Message processGroupMessage(String groupId, Message message);
}
```

### 2.2 Callback应用场景

#### 2.2.1 消息状态回调
```java
public interface MessageCallback {
    // 消息发送结果回调
    void onMessageSent(String msgId, SendResult result);
    
    // 消息送达回调
    void onMessageDelivered(String msgId, List<String> receiverIds);
    
    // 消息已读回调
    void onMessageRead(String msgId, String readerId, long readTime);
}
```

#### 2.2.2 群组事件回调
```java
public interface GroupCallback {
    // 群成员变更回调
    void onMemberChanged(String groupId, GroupMemberChange change);
    
    // 群解散回调
    void onGroupDismissed(String groupId);
    
    // 群公告更新回调
    void onAnnouncementUpdated(String groupId, String announcement);
}
```

#### 2.2.3 第三方系统回调
```java
public interface SystemCallback {
    // 支付结果回调
    void onPaymentResult(String orderId, PaymentResult result);
    
    // 文件上传完成回调
    void onFileUploaded(String fileId, String url);
    
    // 用户状态变更回调
    void onUserStatusChanged(String userId, UserStatus status);
}
```

## 三、配置和使用

### 3.1 Hook配置
```yaml
hooks:
  message:
    - com.im.hook.SensitiveWordHook
    - com.im.hook.AntiSpamHook
  user:
    - com.im.hook.UserAuthHook
    - com.im.hook.UserStatusHook
  group:
    - com.im.hook.GroupPermissionHook
```

### 3.2 Callback配置
```yaml
callbacks:
  message:
    url: http://api.example.com/message/callback
    events:
      - message_sent
      - message_delivered
      - message_read
    retry:
      max_attempts: 3
      interval: 5000
  
  group:
    url: http://api.example.com/group/callback
    events:
      - member_changed
      - group_dismissed
    secret_key: your-secret-key
```

## 四、最佳实践

### 4.1 Hook最佳实践

1. 性能考虑
   - Hook应该快速执行，避免耗时操作
   - 可以使用缓存优化性能
   - 考虑Hook的执行顺序

2. 容错处理
   - Hook异常不应影响主流程
   - 提供默认行为
   - 做好日志记录

3. 扩展性
   - 使用接口定义Hook
   - 支持动态加载
   - 提供优先级机制

### 4.2 Callback最佳实践

1. 可靠性
   - 实现重试机制
   - 保证幂等性
   - 做好失败补偿

2. 安全性
   - 使用HTTPS
   - 加入签名机制
   - 设置超时时间

3. 监控告警
   - 记录调用日志
   - 监控成功率
   - 异常情况告警

## 五、示例代码

### 5.1 Hook示例
```java
@Component
public class MessageHookManager {
    private List<MessageSendHook> hooks;
    
    public boolean executeHooks(Message message) {
        for (MessageSendHook hook : hooks) {
            try {
                if (!hook.beforeMessageSend(message)) {
                    return false;
                }
                message = hook.processMessage(message);
            } catch (Exception e) {
                log.error("Hook执行异常", e);
            }
        }
        return true;
    }
}
```

### 5.2 Callback示例
```java
@Component
public class CallbackManager {
    private final WebClient webClient;
    
    public void sendCallback(String url, Object data) {
        webClient.post()
                .uri(url)
                .bodyValue(data)
                .retrieve()
                .bodyToMono(Void.class)
                .retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(5)))
                .subscribe(
                    null,
                    error -> log.error("Callback失败", error)
                );
    }
}
```

## 六、注意事项

### 6.1 Hook注意事项
1. 避免Hook中包含复杂业务逻辑
2. 注意Hook的执行顺序
3. 做好Hook的性能监控
4. 提供Hook的开关机制
5. 合理设置Hook超时时间

### 6.2 Callback注意事项
1. 确保Callback接口幂等性
2. 实现合适的重试策略
3. 注意回调超时设置
4. 做好回调失败的补偿机制
5. 注意回调接口的安全性

## 七、常见问题

### 7.1 Hook相关问题
1. Q: Hook执行失败如何处理？
   A: 应该有降级策略，并记录错误日志

2. Q: 如何管理多个Hook的执行顺序？
   A: 通过优先级注解或配置文件管理

### 7.2 Callback相关问题
1. Q: 回调失败如何处理？
   A: 实现重试机制，超过重试次数后进入死信队列

2. Q: 如何保证回调的可靠性？
   A: 使用消息队列，实现消息持久化 