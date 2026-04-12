# push-service 详细设计草案

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`push-service` 是 HOS 系统中前端实时通信的唯一受信任入口，负责 WebSocket 连接管理、消息推送、通知中心存储。前端禁止直连 NATS，所有实时消息必须经过本服务转发。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| WebSocket 连接管理 | ✅ | - |
| 连接鉴权与租户隔离 | ✅ | - |
| 消息推送与路由 | ✅ | - |
| 通知中心存储 | ✅ | - |
| 实时状态同步 (缓存失效) | ✅ | - |
| 连接限流与过载保护 | ✅ | - |
| 消息持久化（离线未读）| ✅ | - |
| 业务事件订阅管理 | ✅ | - |
| NATS 直接消费 | ❌ | 前端禁止直连 |
| 复杂业务逻辑处理 | ❌ | 由各业务服务负责 |

### 1.2 核心约束

- **多租户隔离**: 所有 WebSocket 连接必须携带租户上下文
- **安全准入**: 所有 WebSocket 握手必须验证业务 JWT
- **主题白名单**: 订阅主题必须受平台收口，禁止前端传入任意 Subject
- **连接限流**: 单租户/单用户并发连接数限制
- **过载保护**: 消息队列积压时触发背压策略
- **审计日志**: 所有连接、订阅、推送操作必须记录

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### websocket_connections (WebSocket 连接表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| user_id | VARCHAR | 用户 ID | IDX |
| connection_id | VARCHAR | 连接 ID (唯一) | UNIQUE, IDX |
| client_type | VARCHAR | 客户端类型 (web/mobile) | IDX |
| ip_address | VARCHAR | 客户端 IP | - |
| user_agent | TEXT | User Agent | - |
| connected_at | TIMESTAMP | 连接时间 | IDX |
| last_pong_at | TIMESTAMP | 最后心跳时间 | IDX |
| status | VARCHAR | 状态 (active/disconnected/forced_closed) | IDX |
| disconnect_reason | VARCHAR | 断开原因 | - |
| metadata | JSONB | 附加元数据 | - |

**索引策略**:
- `(tenant_id, connection_id)` - 连接查询
- `(tenant_id, user_id)` - 用户连接历史
- `(tenant_id, status, last_pong_at DESC)` - 心跳超时检查

#### subscriptions (订阅主题表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| connection_id | VARCHAR | 连接 ID | FK, IDX |
| subject | VARCHAR | 订阅主题 | IDX |
| subject_type | VARCHAR | 主题类型 (user/role/broadcast) | IDX |
| subscribe_at | TIMESTAMP | 订阅时间 | IDX |
| unsubscribe_at | TIMESTAMP | 取消订阅时间 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**主题收口**:
```
# 用户级主题 (只能订阅自己的)
t.{tenant_id}.notify.user.{user_id}

# 角色级主题 (同角色用户共享)
t.{tenant_id}.notify.role.{role}

# 广播主题 (所有连接用户)
t.{tenant_id}.notify.broadcast
```

**索引策略**:
- `(tenant_id, connection_id, subject)` - 订阅查询
- `(tenant_id, subject_type, unsubscribe_at IS NULL)` - 有效订阅查询

#### notifications (通知中心表，离线未读消息)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| user_id | VARCHAR | 接收用户 ID | IDX |
| connection_id | VARCHAR | 连接 ID (发送时) | FK, IDX |
| notification_type | VARCHAR | 通知类型 | IDX |
| subject | VARCHAR | 消息主题 | - |
| title | TEXT | 消息标题 | - |
| content | TEXT | 消息内容 | - |
| data | JSONB | 消息数据 | - |
| read_at | TIMESTAMP | 已读时间 | IDX |
| expires_at | TIMESTAMP | 过期时间 | IDX |
| priority | VARCHAR | 优先级 (critical/normal/low) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |
| sent_at | TIMESTAMP | 发送时间 | IDX |

**通知类型**:
| 类型 | 说明 |
|------|------|
| appointment_created | 预约创建成功 |
| appointment_cancelled | 预约取消 |
| appointment_checked_in | 患者签到 |
| appointment_started | 就诊开始 |
| appointment_completed | 就诊完成 |
| payment_success | 支付成功 |
| payment_failed | 支付失败 |
| imaging_ai_processed | AI 影像分析完成 |
| system_announcement | 系统公告 |
| queue_update | 排队状态更新 |

**索引策略**:
- `(tenant_id, user_id, read_at IS NULL)` - 用户未读消息查询
- `(tenant_id, user_id, expires_at, read_at IS NULL)` - 未过期未读消息查询
- `(tenant_id, created_at DESC)` - 最新消息查询

#### message_queue (推送队列，离线消息存储)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| notification_id | VARCHAR | 关联通知 ID | FK, IDX |
| status | VARCHAR | 状态 (pending/sent/failed) | IDX |
| retry_count | INT | 重试次数 | IDX |
| next_retry_at | TIMESTAMP | 下次重试时间 | IDX |
| error_message | TEXT | 错误信息 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, status, next_retry_at)` - 待重试消息查询

#### tenant_rate_limits (租户限流配置表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| tenant_id | VARCHAR | 租户 ID | FK, UNIQUE, IDX |
| max_connections_per_user | INT | 单用户最大连接数 | - |
| max_connections_total | INT | 全局最大连接数 | - |
| max_messages_per_minute | INT | 每分钟最大消息数 | - |
| queue_depth_limit | INT | 队列深度限制 | - |
| backpressure_threshold | INT | 背压阈值 | - |
| created_at | TIMESTAMP | 创建时间 | - |
| updated_at | TIMESTAMP | 更新时间 | - |

#### outbox_events (Outbox 模式)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | IDX |
| aggregate_type | VARCHAR | 聚合类型 | IDX |
| aggregate_id | VARCHAR | 聚合 ID | IDX |
| event_type | VARCHAR | 事件类型 | - |
| event_data | JSONB | 事件数据 | - |
| published | BOOLEAN | 是否已发布 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

---

## 3. API 接口设计

### 3.1 WebSocket 握手

#### WS /ws/v1/connect

WebSocket 握手，携带业务 JWT

**握手请求 (Header)**:
```
Authorization: Bearer {business_jwt}
X-Client-Type: web/mobile
X-Device-Id: {device_id}
```

**握手响应**:
```json
{
  "connection_id": "conn_abc123",
  "tenant_id": "tenant_999",
  "user_id": "user_123",
  "user_roles": ["patient"],
  "server_timestamp": 1712880000
}
```

### 3.2 消息推送

服务端主动推送到客户端

#### WS /ws/v1/push

推送消息到指定连接

```json
{
  "type": "notification",
  "event_id": "evt_abc123",
  "notification_id": "notif_abc123",
  "notification_type": "appointment_checked_in",
  "data": {
    "appointment_id": "apt_123",
    "patient_id": "patient_123",
    "queue_position": 3,
    "estimated_wait_minutes": 10
  }
}
```

#### WS /ws/v1/broadcast

广播消息到指定主题

```json
{
  "type": "broadcast",
  "subject": "t.tenant_999.notify.role.doctor",
  "event_id": "evt_abc123",
  "data": {
    "message": "新预约已创建",
    "appointment_id": "apt_123"
  }
}
```

### 3.3 订阅管理

#### WS /ws/v1/subscribe

订阅主题

**请求**:
```json
{
  "subjects": [
    "t.tenant_999.notify.user.123",
    "t.tenant_999.notify.role.doctor"
  ]
}
```

**响应**:
```json
{
  "subscribed": ["t.tenant_999.notify.user.123"],
  "failed": [],
  "reasons": []
}
```

#### WS /ws/v1/unsubscribe

取消订阅主题

**请求**:
```json
{
  "subjects": ["t.tenant_999.notify.user.123"]
}
```

### 3.4 心跳机制

#### WS /ws/v1/ping

客户端发送心跳

**请求**:
```json
{
  "timestamp": 1712880000
}
```

**响应**:
```json
{
  "pong": true,
  "server_timestamp": 1712880000
}
```

### 3.5 REST API (用于离线消息查询)

#### GET /api/v1/notifications/unread

获取未读消息数量

**响应**:
```json
{
  "unread_count": 5,
  "unread_critical_count": 1
}
```

#### GET /api/v1/notifications

分页查询消息列表

**查询参数**:
- `page`: 页码
- `page_size`: 每页数量
- `read_status`: all/unread/read
- `notification_type`: 通知类型过滤

**响应**:
```json
{
  "total": 20,
  "page": 1,
  "page_size": 10,
  "items": [
    {
      "id": "notif_abc123",
      "type": "appointment_checked_in",
      "title": "您已签到",
      "content": "当前排队第 3 位",
      "read": false,
      "created_at": "2026-04-15T09:02:00Z"
    }
  ]
}
```

#### POST /api/v1/notifications/:id/read

标记消息为已读

#### DELETE /api/v1/notifications/:id

删除消息

---

## 4. 连接管理与心跳

### 4.1 连接生命周期

```
1. WebSocket 握手 (验证 JWT)
   ↓
2. 创建连接记录
   ↓
3. 发送连接成功确认
   ↓
4. 心跳保活 (每 30 秒)
   ↓
5. 心跳超时检测 (2 次未响应)
   ├─ 发送 Ping
   ├─ 延迟 30 秒等待 Pong
   └─ 超时断开连接
   ↓
6. 正常/异常断开
   ├─ 清理订阅
   ├─ 通知业务服务（连接断开事件）
   └─ 保留未发送消息到通知中心
```

### 4.2 心跳机制

```
客户端                          服务端
   │                              │
   │◄──── Ping ──────┤            │
   │                     └──── Pong ──│
   │         (30 秒间隔)              │
   └───────────────────────────────────┘
```

**心跳超时检测**:
- 客户端每 30 秒发送 Ping
- 服务端记录 `last_pong_at`
- 2 分钟未收到 Pong → 主动断开连接

### 4.3 背压策略

当消息队列积压超过阈值时：

1. **轻度积压** (queue_depth_limit * 0.8):
   - 通知所有连接：队列积压，消息可能延迟
   - 延迟消息（广播/低优先级）

2. **中度积压** (queue_depth_limit * 1.2):
   - 暂停广播主题
   - 只处理高优先级消息
   - 拒队新消息

3. **重度积压** (queue_depth_limit * 2.0):
   - 暂停新消息入队
   - 扩容 worker 数量
   - 告警

---

## 5. 消息路由与权限

### 5.1 主题收口

| 主题格式 | 订阅条件 | 典型场景 |
|---------|---------|---------|
| `t.{tenant_id}.notify.user.{user_id}` | 用户只能订阅自己的主题 | 个人通知 |
| `t.{tenant_id}.notify.role.{role}` | 同角色用户共享 | 医生/护士群发 |
| `t.{tenant_id}.notify.broadcast` | 所有用户可订阅（需审批） | 系统公告 |

**白名单规则**:
- 主题必须匹配上述格式
- 禁止前端传入任意 Subject
- 动态部分 {user_id}/{role} 必须经过验证

### 5.2 消息类型

| 类型 | 方向 | 优先级 |
|------|------|--------|
| notification | 服务端 → 客户端 | 可配置 |
| command | 客户端 → 服务端 | 正常优先级 |
| cache_invalidation | 服务端 → 客户端 | 高优先级 |
| heartbeat | 双向 | 最低优先级 |

### 5.3 权限控制

| 操作 | 验证方式 |
|------|---------|
| 建立连接 | JWT 验证 + user_roles |
| 订阅主题 | 主题白名单 + user_id 匹配 |
| 发送广播 | 需要 admin 权限 |
| 发送到用户主题 | 需要 admin 权限或发送者与目标用户同角色 |

---

## 6. NATS 集成

### 6.1 订阅 NATS JetStream

`push-service` 作为 NATS 消费者，订阅业务服务发布的事件：

```
booking-service:
  t.{tenant_id}.events.appointment
  t.{tenant_id}.events.booking_status

emr-service:
  t.{tenant_id}.events.encounter
  t.{tenant_id}.events.imaging_study

payment-service:
  t.{tenant_id}.events.transaction
  t.{tenant_id}.events.refund
```

### 6.2 事件转 WebSocket 消息

| NATS 事件 | WebSocket 消息类型 | 通知类型 |
|---------|--------------|----------|
| appointment.created | notification | appointment_created |
| appointment.checked_in | notification | appointment_checked_in |
| appointment.started | cache_invalidation | appointment_started |
| appointment.completed | notification | appointment_completed |
| payment.paid | notification | payment_success |
| imaging_study.uploaded | notification | imaging_ai_processed |

---

## 7. 离线消息处理

### 7.1 离线检测流程

```
1. 业务服务发布事件 (如 appointment.created)
   ↓
2. push-service 查询用户在线连接
   ├─ 有连接: 直接推送
   └─ 无连接: 存入 notifications (离线未读)
   ↓
3. 用户重新连接时
   └─ 批量推送离线未读消息
```

### 7.2 消息过期清理

```
每日 03:00 定时任务
   ↓
1. 标记过期消息 (read_at IS NULL AND expires_at < NOW)
   ↓
2. 软删除 7 天前的已读消息
   ↓
3. 硬删除 30 天前的所有消息
```

---

## 8. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `booking-service` | gRPC | 订阅预约事件 |
| `emr-service` | gRPC | 订阅就诊、影像事件 |
| `payment-service` | gRPC | 订阅支付、退款事件 |
| `NATS JetStream` | 直接连接 | 事件订阅 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `Redis` | 直接连接 | 连接状态缓存、队列缓存 |

---

## 9. 安全与合规

### 9.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 建立连接 | any | `push:connect` |
| 订阅用户主题 | user, admin | `push:subscribe_user` |
| 订阅角色主题 | user, admin | `push:subscribe_role` |
| 发送广播 | admin | `push:broadcast` |
| 查看未读消息 | patient, admin | `push:read_notifications` |
| 标记消息已读 | patient | `push:mark_read` |
| 删除消息 | patient, admin | `push:delete_notification` |

### 9.2 安全措施

- **JWT 验证**: 所有连接必须携带有效的业务 JWT
- **租户隔离**: 连接必须携带 tenant_id，跨租户消息被阻止
- **主题白名单**: 严格验证主题格式，禁止任意订阅
- **IP 限流**: 单 IP 短时间内连接数限制
- **消息加密**: 敏感数据在传输前加密
- **审计日志**: 所有连接、订阅、推送操作记录

### 9.3 防攻击措施

| 攻击类型 | 防御措施 |
|---------|---------|
| 连接风暴 | 连接限流 + IP 黑名单 |
| 消息炸弹 | 每分钟最大消息数限制 |
| 未授权订阅 | 主题白名单验证 |
| 跨租户攻击 | tenant_id 隔离验证 |
| 重放攻击 | JWT 有效期 + 连接超时 |

---

## 10. 性能指标

| 指标 | 目标值 |
|------|--------|
| 连接建立时间 | < 100ms |
| 消息推送延迟 | < 50ms (P99) |
| 心跳响应时间 | < 20ms |
| 并发连接数 | > 10,000 |
| 消息吞吐量 | > 50,000 msg/s |
| 消息丢失率 | < 0.01% |

---

## 11. 待讨论事项

1. **消息持久化策略**: 离线消息保留多久？（建议：未读 30 天，已读 7 天）
2. **广播消息审批**: 是否需要审批流程才能发送系统公告？
3. **消息撤回**: 是否支持消息撤回（发送后一定时间内）？
4. **富媒体消息**: 是否支持图片、语音、视频消息？
5. **消息确认**: 是否需要消息已送达确认机制？
6. **群发优化**: 大量用户群发是否需要分批处理？

---

@gemini @gpt52
以上是 `push-service` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: WebSocket 心跳动效、消息气泡 UX、通知中心视觉布局是否需要补充？
- **@gpt52**: 背压策略是否完整？连接限流是否安全？主题白名单验证是否严格？是否有安全风险？
