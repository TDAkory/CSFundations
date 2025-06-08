# Chat system

![chat system](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CSFundations/chat_system.png)

## 需求定义

### 功能需求（Functional Requirement）

- 1对1聊天（1-1 chat）：支持在线消息投递（online message delivery）、离线消息（support offline message）
- 群聊（group chat）：人数限制 `<200` 
- 状态显示（presence status） 

### 非功能需求（Non-Functional Requirements）

- 可扩展性（scalability）：支持10亿（1B）用户  
- 低延迟（low latency）： `<500ms`  
- 高一致性（high consistency）  
- 容灾/故障转移（FT）  

## 技术架构

### 通信协议选项

polling（轮询）、long polling（长轮询）、websocket（WebSocket）、QUIC

### 架构组件与交互

- **客户端**：Client A、Client B、Client C  
- **服务端**：Chat Server（多实例）  
- **核心中间件**：  
  - Router（路由，连接 Pub/Sub、Session Registration 等模块）  
  - Pub/Sub（发布-订阅，处理消息广播）  
  - Group Registration（群聊注册，管理群组信息）  
  - Session Registration（会话注册，管理用户在线状态）  
  - Offline Storage（离线存储，保存未投递的离线消息）  

```plantuml
@startuml  
title Chat System Component Diagram  

package "Client Layer" {  
  component "Client A" as ClientA  
  component "Client B" as ClientB  
}  

package "Access Layer" {  
  component "Chat Server" as ChatServer  
}  

package "Routing & Distribution" {  
  component "Router" as Router  
  component "Pub/Sub" as PubSub  
  component "Session Registration" as SessionReg  
}  

// 交互关系  
ClientA --> ChatServer : 消息请求  
ClientB --> ChatServer : 消息请求  
ChatServer --> Router : 转发消息  
Router --> PubSub : 发布/订阅  
Router --> SessionReg : 会话注册/查询  

@enduml  
```

### 容量与性能估算（部分示例）

- 10亿用户 * 20% 活跃 = 2亿（200M）  
- 2亿 * 25% 在线 = 5000万（50M）  
- 消息吞吐量：`0.2 msg/sec * 50M = 10M msg/s`  
- 带宽估算：`10M * 0.5KB = 5GB/s`  
- 服务器规模：`50M / 2M ≈ 25 server`（单服务器承载 2M 在线用户）  

### 消息结构定义

- **单聊消息（message）**：  

```shell
message {  
  message_id - 8bytes  
  from - 8byte  
  to - 8byte  
  content - 600-800 bytes  
  timestamp - 8 bytes  
}  
// 单条大小：<1KB（约 0.5KB）  
```

- **群聊消息（group message）**：
  
```shell
group message {  
  group_id  
  message_id  
  user_id  
  content  
  timestamp  
}  
```

### 六、技术方案补充

- **通信协议**：polling、long polling、websocket、QUIC（可选）  
- **数据模型**：用户状态存储（`user_id, last_heartbeat: timestamp, chat_server: xxx`）  
- **优化策略**：session affinity（会话亲和性）、message sync（消息同步）、edge aggr + center sync（边缘聚合 + 中心同步）  
