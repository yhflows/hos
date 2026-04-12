# BFF (Backend for Frontend) 详细设计草案

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

BFF 层是 HOS 系统的统一接入层，负责前端协议适配、业务 JWT 签发、多租户上下文注入、限流熔断。前端禁止直接访问后端微服务，所有请求必须经过 BFF 路由。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 用户认证代理 | ✅ | - |
| 业务 JWT 签发 | ✅ | - |
| 请求聚合 | ✅ | - |
| 多租户上下文注入 | ✅ | - |
| 限流与熔断 | ✅ | - |
| 前端协议适配 | ✅ | - |
| 业务逻辑编排 | ❌ | 由各微服务负责 |
| 数据持久化 | ❌ | 直接由微服务访问数据库 |
| NATS 订阅 | ❌ | push-service 直连 |

### 1.2 核心约束

- **单一入口原则**: 前端只能访问 BFF，禁止直连后端微服务
- **安全边界**: 所有后端调用必须携带业务 JWT
- **多租户隔离**: 所有请求必须在租户 schema 内执行
- **限流保护**: 防止恶意或过度调用导致系统过载
- **熔断降级**: 后端服务异常时自动降级，避免雪崩
- **审计日志**: 所有外部调用必须记录审计日志

---

## 2. BFF 分层架构

```
前端
    │
    ├─ admin-bff (管理后台 BFF)
    ├─ patient-bff (患者端 BFF)
    └─ wechat-bff (微信生态适配 BFF)
         │
         ├─ 微信小程序 (Taro)
         ├─ 微信公众号
         └─ 微信支付回调
         └─ 微信服务端推送
    │
    └── Traefik (反向代理 + TLS)
             │
             ├─ admin-bff
             ├─ patient-bff
             └─ wechat-bff
```

---

## 3. 数据模型

### 3.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 3.2 核心表结构

#### api_clients (API 客户端表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| client_type | VARCHAR | 客户端类型 (admin/mobile/wechat/web) | IDX |
| client_id | VARCHAR | 客户端 ID (自签名） | UNIQUE, IDX |
| client_secret | VARCHAR | 客户端密钥 | - |
| callback_url | VARCHAR | 回调 URL | - |
| allowed_scopes | JSONB | 允许的权限范围 | - |
| rate_limit_rpm | INT | 限流 (每分钟请求数) | - |
| rate_limit_burst | INT | 限流 (峰值) | - |
| is_active | BOOLEAN | 是否启用 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, client_type)` - 客户端查询
- `(tenant_id, client_id)` - 唯一约束

#### api_rate_limits (租户级限流配置表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| endpoint_pattern | VARCHAR | 端点模式 (如 `/api/v1/booking/*`) | IDX |
| rate_limit_rpm | INT | 限流 (每分钟请求数) | - |
| rate_limit_burst | INT | 限流 (峰值) | - |
| algorithm | VARCHAR | 算法 (token-bucket/leaky-bucket/fixed-window) | IDX |
| priority | INT | 优先级 (1-10) | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

#### circuit_breakers (熔断器配置表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| breaker_name | VARCHAR | 熔断器名称 | IDX |
| target_service | VARCHAR | 目标服务名 (booking/emr/payment) | IDX |
| error_threshold | INT | 错误阈值 (百分比) | - |
| timeout_seconds | INT | 超时时间 (秒) | - |
| half_open_state | VARCHAR | 半开状态 (open/half-open/closed) | - |
| status | VARCHAR | 状态 (active/disabled/forced-open) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**熔断器状态机**:
```
closed → active: 熔断器开始接收请求
active → half-open: 错误率超过阈值，进入半开状态
half-open → open: 所有请求半开，允许慢请求通过
open → closed: 所有请求全开，正常限流
```

#### audit_logs (审计日志表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| user_id | VARCHAR | 用户 ID | IDX |
| client_type | VARCHAR | 客户端类型 | IDX |
| request_method | VARCHAR | 请求方法 (GET/POST/PUT/DELETE) | - |
| request_path | TEXT | 请求路径 | - |
| response_status | INT | 响应状态 (200/400/500/503) | - |
| response_time_ms | INT | 响应时间 (毫秒) | - |
| upstream_service | VARCHAR | 上游服务 | IDX |
| business_jwt_id | VARCHAR | 业务 JWT ID (用于追踪) | - |
| error_message | TEXT | 错误信息 | - |
| ip_address | VARCHAR | IP 地址 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, user_id, created_at DESC)` - 用户调用历史
- `(tenant_id, upstream_service, error_message)` - 错误查询

---

## 4. 认证与授权

### 4.1 双令牌模型

- **Casdoor Token**: 用户身份验证，仅用于获取业务 JWT
- **Business JWT**: 业务访问令牌，由 BFF 签发，包含租户和权限信息

### 4.2 登录流程

```
1. 前端提交账号密码、短信验证码或微信登录 Code
   ↓
2. BFF 转发请求到 Casdoor
   ↓
3. Casdoor 返回用户信息和组织关系
   ↓
4. BFF 验证租户状态并选择当前租户
   ↓
5. BFF 签发业务 JWT (携带 tenant_id, user_id, roles, scope)
```

### 4.3 业务 JWT 声明

```json
{
  "sub": "user_12345",
  "tenant_id": "tenant_999",
  "roles": ["doctor"],
  "scope": ["booking:read", "emr:write"],
  "token_version": 1,
  "exp": 1712880000,
  "iss": "bff",
  "aud": ["https://bff.example.com/audit"]
}
```

### 4.4 租户选择策略

| 场景 | 选择策略 |
|------|---------|
| 单租户 | 默认选择唯一租户 |
| 多点执业 | 用户需手动切换租户，BFF 仅验证是否有权限 |
| 管理员模式 | 管理员可访问所有租户（需特殊权限） |

---

## 5. 服务间通信

### 5.1 gRPC 调用约定

| 服务 | 通信方式 | 目的 |
|------|---------|------|
| `clinic-service` | gRPC | 查询医生、科室、排班 |
| `booking-service` | gRPC | 预约 CRUD、号源查询 |
| `emr-service` | gRPC | 病历 CRUD、牙位图、影像 |
| `payment-service` | gRPC | 账单、交易、退款 |
| `push-service` | gRPC | 发送通知 |
| `supplychain-bff` | gRPC | Odoo 适配 |
| `weichat-bff` | gRPC | 微信支付回调、消息推送 |

### 5.2 Outbox 事件

| 事件类型 | 发布服务 | 消费方 |
|---------|---------|---------|
| `tenant.selected` | BFF | push-service (通知用户租户切换) |
| `user.login` | BFF | push-service (通知用户登录) |
| `subscription.changed` | BFF | push-service (通知用户订阅变更) |
| `rate.limit.exceeded` | BFF | push-service (通知限流触发) |
| `circuit.opened` | BFF | push-service (通知熔断器打开) |
| `audit.log` | BFF | 审计服务 (记录所有 API 调用) |

---

## 6. 多租户上下文注入

### 6.1 注入策略

| 层级 | 注入方式 | 说明 |
|------|---------|------|
| HTTP Headers | `X-Tenant-ID`, `X-User-ID` | 全局注入 |
| HTML Metadata | `<meta name="tenant-theme" content="...">` | 前端可读取 |
| API Response | 遵循 ADR-003，只返回白名单允许的 CSS 变量 | 租户主题配置 |

### 6.2 租户主题配置 (遵循 ADR-003)

**获取接口**:
```
GET /api/v1/tenant/theme
```

**响应**:
```json
{
  "primary_color": "#1890ff",
  "border_radius": "8px",
  "font_scale": "1.0",
  "logo_url": "https://...",
  "contrast_min": 4.5,
  "contrast_max": 7.0
}
```

---

## 7. API 接口设计

### 7.1 管理后台 (admin-bff)

#### 登录相关

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/auth/login` | POST | 管理员登录 |
| `/api/v1/auth/logout` | POST | 退出登录 |
| `/api/v1/auth/tenant/switch` | POST | 切换租户 |

#### 预约挂号

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/booking/schedules/available` | GET | 查询可用号源 |
| `/api/v1/booking/appointments` | GET | 查询预约列表 |
| `/api/v1/booking/appointments` | POST | 创建预约 |
| `/api/v1/booking/appointments/:id` | GET | 获取预约详情 |
| `/api/v1/booking/appointments/:id/cancel` | POST | 取消预约 |
| `/api/v1/booking/appointments/:id/check-in` | POST | 签到 |

#### 电子病历 (emr-service 代理)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/emr/encounters` | GET | 就诊记录列表 |
| `/api/v1/emr/encounters` | POST | 创建就诊记录 |
| `/api/v1/emr/encounters/:id` | GET | 获取就诊详情 |
| `/api/v1/emr/encounters/:id/notes` | GET | 获取临床笔记 |
| `/api/v1/emr/tooth-charts/:id` | GET | 获取牙位图 |
| `/api/v1/emr/imaging/studies` | GET | 获取影像列表 |

#### 支付 (payment-service 代理)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/payment/bills` | GET | 获取账单 |
| `/api/v1/payment/bills/:encounter-id` | GET | 获取就诊账单 |
| `/api/v1/payment/transactions` | POST | 创建交易 |
| `/api/v1/payment/transactions/:id` | GET | 获取交易详情 |
| `/api/v1/payment/refunds` | POST | 申请退款 |
| `/api/v1/payment/refunds/:id/approve` | POST | 审批退款 |

#### 通知 (push-service 代理)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/notifications/unread-count` | GET | 获取未读数量 |
| `/api/v1/notifications` | GET | 分页获取通知 |
| `/api/v1/notifications/:id/read` | POST | 标记已读 |
| `/api/v1/notifications/:id/delete` | DELETE | 删除通知 |

### 7.2 患者端 (patient-bff)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/patient/login` | POST | 患者登录 |
| `/api/v1/patient/register` | POST | 患者注册 |
| `/api/v1/patient/appointments` | GET | 查询患者预约 |
| `/api/v1/patient/imaging` | GET | 查询患者影像 |
| `/api/v1/patient/emr` | GET | 查询患者病历 |

### 7.3 微信适配 (wechat-bff)

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/wechat/jsapi/ticket` | POST | 创建微信 JSAPI Ticket |
| `/api/v1/wechat/subscribe` | POST | 订阅模板消息 |
| `/api/v1/wechat/callback/payment` | POST | 微信支付回调 |

---

## 8. 限流与熔断

### 8.1 限流算法

| 算法 | 说明 | 使用场景 |
|------|------|------|---------|
| token-bucket | 令牌桶 | 限制调用频率（用户级） |
| fixed-window | 固定窗口 | 滑动窗口限流（租户/端点） |
| leaky-bucket | 漏桶 | 短期限流（突发流量） |

### 8.2 限流实现

```
Redis 数据结构:
rate_limit:{tenant_id}:{user_id}:{endpoint} = remaining_count

每次请求前:
1. 检查 Redis 限流配置
2. 如果达到限制，返回 HTTP 429 Too Many Requests
3. 请求成功后递减计数
4. 计数器使用滑动窗口算法自动恢复
```

### 8.3 熔断实现

```
Redis 数据结构:
circuit_state:{tenant_id}:{breaker_name}:{service} = {state, failure_count, last_failure_time}

每次请求:
1. 检查熔断器状态
2. 如果状态为 open，直接转发请求
3. 如果状态为 half-open，放行 50% 请求
4. 如果状态为 closed，直接拒绝
5. 记录响应状态和耗时
6. 连续成功 5 次后转为 open，连续失败 3 次后转为 closed
```

---

## 9. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `Casdoor` | HTTP | 用户身份验证 |
| `clinic-service` | gRPC | 医生、科室、排班 |
| `booking-service` | gRPC | 预约 CRUD |
| `emr-service` | gRPC | 电子病历、影像 |
| `payment-service` | gRPC | 账单、交易、退款 |
| `push-service` | gRPC | 消息推送 |
| `supplychain-bff` | gRPC | Odoo 适配 |
| `weichat-bff` | gRPC | 微信生态 |
| `Redis` | 直接连接 | 限流、熔断状态 |
| `NATS JetStream` | 直接连接 | 发布 Outbox 事件 |
| `PostgreSQL` | 直接连接 | 审计日志 |

---

## 10. 安全与合规

### 10.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 创建交易 | patient, admin | `payment:create_transaction` |
| 申请退款 | patient, admin | `payment:request_refund` |
| 切换租户 | multi-tenant | `tenant:switch` |
| 查看审计日志 | admin | `audit:read` |
| 修改限流配置 | admin | `bff:configure_rate_limit` |
| 查看熔断器状态 | admin | `bff:read_circuit` |

### 10.2 安全措施

- **SQL 注入防护**: 所有数据库查询使用参数化查询，禁止字符串拼接
- **XSS 防护**: 所有用户输入必须经过清理和转义
- **CSRF 防护**: 所有写操作使用 CSRF Token
- **请求签名**: 敏感操作需添加签名验证
- **IP 白名单**: 微信回调端点 IP 白名单验证

---

## 11. 性能指标

| 指标 | 目标值 |
|------|--------|
| BFF 响应时间 (P50) | < 100ms |
| BFF 响应时间 (P99) | < 200ms |
| 熔断响应时间 | < 10ms |
| 限流检查延迟 | < 5ms |
| 认证响应时间 | < 50ms |
| 并发请求处理能力 | > 5,000 RPS |

---

## 12. 待讨论事项

1. **API 版本控制**: 是否采用 OpenAPI/Swagger 规范？是否需要 API 网关？
2. **缓存策略**: 哪一级数据缓存是否需要 Redis 集群缓存？
3. **分布式追踪**: 调用链路追踪方案（OpenTelemetry/SkyWalking）？
4. **灰度发布**: 是否需要蓝绿/金丝雀发布策略？
5. **监控大盘**: Grafana 监控指标如何定义？

---

@gemini @gpt52
以上是 `bff-service` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: 认证流程 UX 是否清晰？租户切换界面、限流提示、熔断降级用户提示是否需要补充？
- **@gpt52**: 限流算法是否完整？熔断器状态机是否健壮？审计日志是否完整？是否有安全风险？
