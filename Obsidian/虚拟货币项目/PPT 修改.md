在线系统涉及的一些名词和解释：

| 名词    | 解释                                  |
| ----- | ----------------------------------- |
| 系统管理员 | 拥有对在线系统的用户管理权限和账户管理权限，可以修改系统中人员的角色。 |
| 授权用户  | 可以对在线系统中的任意账户发起交易请求，在离线系统中完成签名。     |
| 审计用户  | 负责审计在线系统中所有的操作记录和交易记录。              |
|       |                                     |

## 1 在线存管提控系统实现的功能

系统采用 B/S（Browser/Server）架构，在服务端部署独立的 Geth 客户端，并依赖于离线签名系统进行交易签名。

交易流程为：
- 授权用户发起交易请求
- 服务器验证并准备交易数据
- 发送至离线系统进行签名
- 接收签名后执行交易并广播到以太坊网络
- 记录交易信息到数据库

```mermaid
graph TB
    subgraph Browser["浏览器"]
        UI["用户界面"]
    end

    subgraph Server["服务器"]
        API["API服务"]
        Auth["认证授权"]
        
        subgraph Database["数据库"]
            UserTable["用户表"]
            AccountTable["账户表"]
            TxTable["交易表"]
            OpLogTable["操作表"]
        end
        
        subgraph Blockchain["区块链模块"]
            Geth["Geth客户端"]
            TxBroadcast["交易广播"]
        end
    end
    
    subgraph Ethereum["以太坊网络"]
        ETH["以太坊节点"]
    end
    
    subgraph Offline["离线签名系统"]
        Sign["交易签名"]
    end

    UI -->|HTTP请求| API
    API -->|认证| Auth
    API -->|CRUD| Database
    Browser -->|发送待签名交易| Offline
    Offline -->|返回签名后交易| Browser
    API -->|广播交易| TxBroadcast
    TxBroadcast -->|发送交易| Geth
    Geth <-->|同步| ETH
```

## 2 技术选型软硬件条件

在线系统：
需要一台服务器，运行 Linux 环境，2 核 CPU、2GB 内存以上、30GB SSD 以上，能连接互联网即可。

离线系统：
需要一台服务器，运行 Linux 环境，2 核 CPU、2GB 内存以上、30GB SSD 以上。不联网。
安全芯片，支持 PC/SC 标准库驱动和 APDU 命令， 3 个及以上。
U 盘，在线系统和离线系统之间传递数据的物理介质，