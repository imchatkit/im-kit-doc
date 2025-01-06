# API接口文档

## 一、认证接口

### 1.1 获取Token
```http
POST /v1/token
Content-Type: application/json

{
    "appId": "xxx",
    "secret": "xxx",
    "timestamp": 1234567890,
    "sign": "xxx"
}
```

### 1.2 刷新Token
```http
POST /v1/token/refresh
Authorization: Bearer xxx
```

## 二、消息接口

### 2.1 发送消息
```http
POST /v1/messages
Authorization: Bearer xxx
Content-Type: application/json

{
    "type": 1,
    "to": "uid2",
    "content": {
        "text": "hello"
    }
}
```

### 2.2 撤回消息
```http
DELETE /v1/messages/{msgId}
Authorization: Bearer xxx
```

## 三、群组接口

### 3.1 创建群组
```http
POST /v1/groups
Authorization: Bearer xxx
Content-Type: application/json

{
    "name": "群名称",
    "members": ["uid1", "uid2"]
}
```

### 3.2 群发消息
```http
POST /v1/groups/{groupId}/messages
Authorization: Bearer xxx
Content-Type: application/json

{
    "type": 1,
    "content": {
        "text": "hello"
    }
}
```
```

## 四、用户接口

### 4.1 用户导入
```http
POST /v1/users/import
Authorization: Bearer xxx
Content-Type: application/json

{
    "users": [
        {
            "uid": "user1",
            "name": "张三",
            "avatar": "http://..."
        }
    ]
}
```

### 4.2 获取用户在线状态
```http
GET /v1/users/{uid}/status
Authorization: Bearer xxx
``` 