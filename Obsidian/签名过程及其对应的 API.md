# 签名过程及其对应的 API

1. **授权用户**输入 $(fromAddr,toAddr,Amount)$，表示 A 要给 B 转多少钱。（登录过程我省略了）
2. 服务器打包交易数据并对其进行哈希运算，返回 $(Hash,fromAddr)$ 给授权用户并将 $(fromAddr, TxData, Hash)$ 存储在 $TxTable$ 表中。（这里主要用到的是交易表，和用户表关系不大）
3. 授权用户将 $(Hash,fromAddr)$ 通过物理介质拷贝到离线系统
4. 离线系统的系统管理员输入 $(Hash,fromAddr)$ 到 $Server$（也是省略了登录认证过程）
5. $Server$ 根据 $fromAddr$ 会找到管理这个地址的三个案件管理员和对应的三个安全芯片和分别对应的数据。创建 $signTable$ 表：$(randNum_i，fromAddr，Hash，userID_i，SN_i，normalData_i，Parties)$。
6. $Server$ 给案件管理员 $userID_i$ 发送通知，通知内容为 $randNum_i$。
7. **案件管理员**登录 $Client$，除 $userID$ 和 $PIN$ 外，还需要需要携带 $randNum_i$。
8. $Client$ 和 $Server$ 建立安全通道后，将 $randNum_i$ 发送回 $Server$。
9. $Server$ 端首先验证 $userID$，$SN$ 是不是和 $signTable$ 中的一致，然后发送 $(Hash,normalData,Parties)$ 到 $Client$，并将 $SE$ 上对应的 $secretData$ 设为可用。
10. $normalData + secretData => Data$，使用数据 $(Hash,Parties,Data)$ 进行签名得到 $Sign$。
11. $Client$ 发送 $(randNum_i,Sign)$ 给 $Server$。
12. $Server$ 通过 $randNum_i$ 从 $signTable$ 中找到 $fromAddr$，打包数据 $(Sign, fromAddr)$ 给 $Browser$。
13. 将 $(Sign,fromAddr)$ 通过物理介质拷贝到在线系统的 $Browser$。
14. 授权用户将 $(Sign,fromAddr)$ 发送到 $Server$ 上，$Server$ 查找 $TxTable$ 获取 $TxData$，将交易上链。
15. 交易成功/失败。

## 1 在线系统 API

### 1.1 创建交易请求

**接口说明**: 授权用户发起转账交易请求

**请求 URL**: `/api/online/transaction/create`

**请求方式**: POST

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|fromAddr|String|是|转出地址|
|toAddr|String|是|转入地址|
|amount|Number|是|转账金额|

**请求示例**:

```json
{
  "fromAddr": "0x1234567890abcdef1234567890abcdef12345678",
  "toAddr": "0xabcdef1234567890abcdef1234567890abcdef12",
  "amount": 100.5
}
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|hash|String|交易哈希|
|fromAddr|String|转出地址|
|status|String|交易状态|

**响应示例**:

```json
{
  "hash": "0xf7b8c12a4e76b1a34c4ea39c5177365493eb2e14850c069d678d1375a21cb81c",
  "fromAddr": "0x1234567890abcdef1234567890abcdef12345678",
  "status": "PENDING_SIGNATURE"
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器内部错误

### 1.2 提交签名上链

**接口说明**: 授权用户提交签名结果，将交易上链

**请求 URL**: `/api/online/transaction/submit`

**请求方式**: POST

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|sign|String|是|签名结果|
|fromAddr|String|是|转出地址|

**请求示例**:

```json
{
  "sign": "0xd8da6bf26964af9d7eed9e03e53415d37aa96045e643e8f94dc0f5bed75c5b..."
  "fromAddr": "0x1234567890abcdef1234567890abcdef12345678"
}
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|txId|String|链上交易 ID|
|status|String|交易状态：SUCCESS/FAILED|
|message|String|状态描述信息|

**响应示例**:

```json
{
  "txId": "0xd8da6bf26964af9d7eed9e03e53415d37aa96045e643e8f94dc0f5bed75c5b...",
  "status": "SUCCESS",
  "message": "交易已成功上链"
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器内部错误

### 1.3 查询交易

**接口说明**: 查询交易状态和详情

**请求 URL**: `/api/online/transaction/query`

**请求方式**: GET

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|hash|String|否|交易哈希|
|fromAddr|String|否|转出地址|
|status|String|否|交易状态|
|page|Number|否|页码 (从 1 开始)|
|pageSize|Number|否|每页条数|

**请求示例**:

```
/api/online/transaction/query?fromAddr=0x1234567890abcdef1234567890abcdef12345678&page=1&pageSize=10
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|transactions|Array|交易列表|
|total|Number|总条数|

**交易对象结构**:

|参数名|类型|描述|
|---|---|---|
|hash|String|交易哈希|
|fromAddr|String|转出地址|
|toAddr|String|转入地址|
|amount|Number|转账金额|
|status|String|交易状态|
|createTime|String|创建时间|
|updateTime|String|更新时间|

**响应示例**:

```json
{
  "transactions": [
    {
      "hash": "0xf7b8c12a4e76b1a34c4ea39c5177365493eb2e14850c069d678d1375a21cb81c",
      "fromAddr": "0x1234567890abcdef1234567890abcdef12345678",
      "toAddr": "0xabcdef1234567890abcdef1234567890abcdef12",
      "amount": 100.5,
      "status": "SUCCESS",
      "createTime": "2025-03-18T08:30:45Z",
      "updateTime": "2025-03-18T08:45:12Z"
    }
  ],
  "total": 1
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器内部错误

## 2 离线系统 HTTP 请求 API

### 2.1 初始化签名任务

**接口说明**: 离线系统管理员输入交易哈希和地址，初始化签名任务

**请求 URL**: `/api/offline/sign/initiate`

**请求方式**: POST

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|hash|String|是|交易哈希|
|fromAddr|String|是|转出地址|

**请求示例**:

```json
{
  "hash": "0xf7b8c12a4e76b1a34c4ea39c5177365493eb2e14850c069d678d1375a21cb81c",
  "fromAddr": "0x1234567890abcdef1234567890abcdef12345678"
}
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|success|Boolean|是否成功初始化|
|signTasks|Array|签名任务列表|

**签名任务对象结构**:

|参数名|类型|描述|
|---|---|---|
|randNum|String|随机数|
|userID|String|案件管理员 ID|
|status|String|任务状态|

**响应示例**:

```json
{
  "success": true,
  "signTasks": [
    {
      "randNum": "8a7b6c5d4e3f2g1h",
      "userID": "manager001",
      "status": "PENDING"
    },
    {
      "randNum": "1a2b3c4d5e6f7g8h",
      "userID": "manager002",
      "status": "PENDING"
    },
    {
      "randNum": "9i8u7y6t5r4e3w2q",
      "userID": "manager003",
      "status": "PENDING"
    }
  ]
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器内部错误

### 2.2 案件管理员登录

**接口说明**: 案件管理员登录并验证身份

**请求 URL**: `/api/offline/manager/login`

**请求方式**: POST

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|userID|String|是|案件管理员 ID|
|pin|String|是|PIN 码|
|randNum|String|是|随机数|

**请求示例**:

```json
{
  "userID": "manager001",
  "pin": "123456",
  "randNum": "8a7b6c5d4e3f2g1h"
}
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|success|Boolean|登录是否成功|
|token|String|会话令牌|

**响应示例**:

```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySUQiOiJtYW5hZ2VyMDAxIiwicmFuZE51bSI6IjhhN2I2YzVkNGUzZjJnMWgiLCJpYXQiOjE2MTY0MjMyMDAsImV4cCI6MTYxNjQyNjgwMH0.8SZXNd3Jhx9TKQvhXLuPmHuEXW3OFjIV30tQD7vkzHc"
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 403: 禁止访问
- 500: 服务器内部错误

### 2.3 查询签名任务

**接口说明**: 查询签名任务状态

**请求 URL**: `/api/offline/sign/tasks`

**请求方式**: GET

**请求参数**:

|参数名|类型|必选|描述|
|---|---|---|---|
|userID|String|否|案件管理员 ID|
|fromAddr|String|否|转出地址|
|status|String|否|任务状态|
|page|Number|否|页码|
|pageSize|Number|否|每页条数|

**请求示例**:

```
/api/offline/sign/tasks?userID=manager001&status=PENDING&page=1&pageSize=10
```

**响应参数**:

|参数名|类型|描述|
|---|---|---|
|tasks|Array|任务列表|
|total|Number|总条数|

**任务对象结构**:

|参数名|类型|描述|
|---|---|---|
|randNum|String|随机数|
|fromAddr|String|转出地址|
|hash|String|交易哈希|
|userID|String|案件管理员 ID|
|status|String|任务状态|
|createTime|String|创建时间|

**响应示例**:

```json
{
  "tasks": [
    {
      "randNum": "8a7b6c5d4e3f2g1h",
      "fromAddr": "0x1234567890abcdef1234567890abcdef12345678",
      "hash": "0xf7b8c12a4e76b1a34c4ea39c5177365493eb2e14850c069d678d1375a21cb81c",
      "userID": "manager001",
      "status": "PENDING",
      "createTime": "2025-03-18T08:30:45Z"
    }
  ],
  "total": 1
}
```

**状态码**:

- 200: 成功
- 400: 参数错误
- 401: 未授权
- 500: 服务器内部错误

## 3 签名过程 WebSocket API

### 3.1 建立 WebSocket 连接

**接口说明**: 建立 WebSocket 连接以进行签名操作

**WebSocket URL**: `ws://server-domain/api/offline/sign/websocket?token={token}`

**参数说明**:

- token: 通过案件管理员登录 API 获取的会话令牌

### 3.2 WebSocket 消息类型

所有 WebSocket 消息均采用 JSON 格式，并包含 type 字段标识消息类型。

#### 3.2.1 签名请求（服务端 -> 客户端）

```json
{
  "type": "SIGN_REQUEST",
  "data": {
    "hash": "0xf7b8c12a4e76b1a34c4ea39c5177365493eb2e14850c069d678d1375a21cb81c",
    "normalData": "base64编码的普通数据",
    "parties": 3,
    "randNum": "8a7b6c5d4e3f2g1h"
  }
}
```

#### 3.2.2 签名结果（客户端 -> 服务端）

```json
{
  "type": "SIGN_RESULT",
  "data": {
    "randNum": "8a7b6c5d4e3f2g1h",
    "sign": "0xd8da6bf26964af9d7eed9e03e53415d37aa96045e643e8f94dc0f5bed75c5b..."
  }
}
```

#### 3.2.3 签名状态通知（服务端 -> 客户端）

```json
{
  "type": "SIGN_STATUS",
  "data": {
    "randNum": "8a7b6c5d4e3f2g1h",
    "status": "COMPLETED",
    "message": "签名已完成"
  }
}
```

#### 3.2.4 心跳消息（客户端 -> 服务端）

```json
{
  "type": "HEARTBEAT",
  "timestamp": 1616423200000
}
```

#### 3.2.5 心跳响应（服务端 -> 客户端）

```json
{
  "type": "HEARTBEAT_ACK",
  "timestamp": 1616423200100
}
```

#### 3.2.6 错误消息（服务端 -> 客户端）

```json
{
  "type": "ERROR",
  "code": 4001,
  "message": "签名失败：安全芯片未就绪"
}
```

#### 3.2.7 连接关闭消息（服务端/客户端 -> 客户端/服务端）

```json
{
  "type": "CLOSE",
  "reason": "签名任务已完成"
}
```

### 3.3 WebSocket 状态码

|状态码|描述|
|---|---|
|1000|正常关闭|
|1001|终端离开|
|1002|协议错误|
|1003|不可接受的数据类型|
|1008|策略违规|
|1011|服务器内部错误|

## 4 数据结构和状态说明

### 4.1 交易状态（Transaction Status）

|状态值|描述|
|---|---|
|CREATED|交易已创建|
|PENDING_SIGNATURE|等待签名|
|SIGNING|签名中|
|SIGNED|已签名，待上链|
|BROADCASTING|广播中|
|SUCCESS|交易成功|
|FAILED|交易失败|

### 4.2 签名任务状态（Sign Task Status）

|状态值|描述|
|---|---|
|PENDING|等待案件管理员处理|
|PROCESSING|签名处理中|
|COMPLETED|签名完成|
|FAILED|签名失败|
|CANCELED|已取消|

## 5 错误码说明

|错误码|描述|
|---|---|
|1001|参数格式错误|
|1002|必填参数缺失|
|2001|未授权访问|
|2002|令牌过期|
|3001|交易不存在|
|3002|签名任务不存在|
|4001|安全芯片未就绪|
|4002|签名验证失败|
|5001|服务器内部错误|

## 6 签名流程时序图

```
+--------------+      +--------------+      +---------------+      +---------------+
| 授权用户(在线) |      | 在线系统服务端 |      | 离线系统服务端  |      | 案件管理员客户端 |
+--------------+      +--------------+      +---------------+      +---------------+
       |                     |                      |                      |
       | 1.创建交易请求       |                      |                      |
       |-------------------->|                      |                      |
       |                     |                      |                      |
       | 2.返回Hash,fromAddr |                      |                      |
       |<--------------------|                      |                      |
       |                     |                      |                      |
       |            [物理介质传输]                   |                      |
       |------------------------------------------------>|                 |
       |                     |                      |                      |
       |                     |                      | 4.初始化签名任务      |
       |                     |                      |<---------------------|
       |                     |                      |                      |
       |                     |                      | 5.创建签名表并通知    |
       |                     |                      |--------------------->|
       |                     |                      |                      |
       |                     |                      | 7.案件管理员登录      |
       |                     |                      |<---------------------|
       |                     |                      |                      |
       |                     |                      | 8.返回Token          |
       |                     |                      |--------------------->|
       |                     |                      |                      |
       |                     |                      | 9.建立WebSocket连接   |
       |                     |                      |<====================+|
       |                     |                      |                      |
       |                     |                      | 10.发送签名请求数据    |
       |                     |                      |+====================+|
       |                     |                      |                      |
       |                     |                      | 11.进行签名           |
       |                     |                      |<--------------------+|
       |                     |                      |                      |
       |                     |                      | 12.返回签名结果       |
       |                     |                      |+====================+|
       |                     |                      |                      |
       |            [物理介质传输]                   |                      |
       |<------------------------------ ------------|                      |
       |                     |                      |                      |
       | 14.提交签名上链      |                      |                      |
       |-------------------->|                      |                      |
       |                     |                      |                      |
       | 15.返回交易结果      |                      |                      |
       |<--------------------|                      |                      |
       |                     |                      |                      |
```

## 7 开发环境和测试

### 7.1 测试环境

- 在线系统 API 基地址: `https://online-test.example.com/api`
- 离线系统 API 基地址: `https://offline-test.example.com/api`
- WebSocket 基地址: `wss://offline-test.example.com/api/offline/sign/websocket`

### 7.2 生产环境

- 在线系统 API 基地址: `https://online.example.com/api`
- 离线系统 API 基地址: `https://offline.example.com/api`
- WebSocket 基地址: `wss://offline.example.com/api/offline/sign/websocket`

### 7.3 Postman 测试集合

提供 Postman 测试集合文件，可导入 Postman 进行 API 测试： [下载Postman测试集合](https://example.com/postman-collection.json)

## 8 安全要求

1. 所有 HTTP 请求必须使用 HTTPS
2. WebSocket 连接必须使用 WSS
3. 敏感数据传输应进行加密
4. API 访问需要进行身份验证和授权
5. 实施请求限流以防止 DoS 攻击
6. 敏感操作应实施双重验证
7. 所有 API 访问应记录审计日志
