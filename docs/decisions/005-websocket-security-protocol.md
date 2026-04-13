# ADR-005: WebSocket 安全协议与通知模型 (WebSocket Security Protocol)

> 状态: 草案 - 待团队评审
> 创建日期: 2026-04-13
> 作者: @opus (布偶猫)

---

## 1. 状态 (Status)
草案 (Draft) - 待团队评审

## 2. 上下文 (Context)

push-service 需要确保：
- WebSocket 连接鉴权安全
- 消息顺序和幂等性
- 防重放攻击
- 角色变更后立即生效

## 3. 决策结果 (Decision)

### 3.1 消息头字段定义

**要求**: 每条 WebSocket 消息必须包含安全与一致性字段

```json
{
  "message_id": "msg_ulid_123",      // 幂等键，唯一标识
  "seq": 1001,                       // 顺序号，每个 recipient-stream 单调递增
  "occurred_at": "2026-04-13T10:00:00Z",
  "priority": "critical|normal|ephemeral",
  "dedupe_key": "apt_001_cancel",   // 折叠键，同类型消息去重
  "collapse_key": "appointment",         // 堆叠键，UI 层合并展示
  "tenant_id": "tenant_999",
  "user_id": "user_123",
  "type": "notification|command|cache_invalidation|heartbeat",
  "data": { /* 实际负载 */ }
}
```

### 3.2 消息优先级分类

**要求**: 区分消息丢失优先级

| 优先级 | 说明 | 持久化 | 可丢弃 |
|---------|------|---------|--------|
| `critical` | 支付成功、急诊变更、签名完成 | ✅ 永久存储 | ❌ 永不丢 |
| `normal` | 普通业务提醒 | ✅ 入通知中心 | ❌ 可合并保留最后一条 |
| `ephemeral` | 在线状态、弱提示 | ❌ 不持久化 | ✅ 队列满时优先丢 |

### 3.3 断连恢复协议

**要求**: 客户端用 `last_acked_seq` 恢复，服务端按 `seq` 补发

```json
// 客户端重连请求
{
  "reconnect": true,
  "last_acked_seq": 999
}

// 服务端响应
{
  "messages": [
    // 按 seq > 999 的顺序补发
    { "seq": 1000, ... },
    { "seq": 1001, ... }
  ]
}
```

### 3.4 主题订阅白名单验证

**要求**: 客户端只能请求枚举级别的订阅意图

**允许的订阅格式**:
- `user.{user_id}` - 用户只能订阅自己的主题
- `role.{role_name}` - 同角色用户共享
- `broadcast` - 广播主题

**禁止**:
- 客户端传入完整 Subject
- 动态构造 Subject 模式

### 3.5 RBAC 变更主动失效

**要求**: 角色变更、用户踢下线、token 吊销后，现有 WebSocket 连接必须被服务端主动断开

```
触发条件:
1. 用户角色变更 (role_updated)
2. 用户被踢下线 (user_kicked)
3. token 吊销 (token_revoked)
4. 租户冻结 (tenant_suspended)

动作:
- 立即关闭相关 WebSocket 连接
- 发送 reason: "session_invalidated"
```

## 4. 后果 (Consequences)

- 消息有明确的顺序和幂等保证
- 角色变更立即生效，无需等待下次重连
- 高压下 critical 消息不丢失
- 通知投影有足够的字段支持 UI 折叠和堆叠展示
