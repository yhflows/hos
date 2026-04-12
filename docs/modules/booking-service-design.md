# booking-service 详细设计草案

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`booking-service` 负责 HOS 系统的预约挂号、签到分诊和就诊状态管理，是患者与诊所交互的核心入口。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 预约单创建、修改、取消 | ✅ | - |
| 签到/爽约/退号状态流转 | ✅ | - |
| 排班号源管理（号源可用性） | ✅ | - |
| 医生/科室/排班基础数据 | ❌ | clinic-service |
| 支付/退款 | ❌ | payment-service |
| 就诊排队/叫号 | ✅ | 推送到 push-service |
| 电子病历录入 | ❌ | emr-service |

### 1.2 核心约束

- **防超卖**: 号源预约必须保证原子性，同一时段同一医生不能被多人预约
- **多租户隔离**: 所有操作必须在租户 schema 内执行
- **审计日志**: 预约、签到、取消等关键操作必须记录审计日志
- **最终一致性**: 跨服务事件通过 Outbox 模式发布

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### appointments (预约单表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| patient_id | VARCHAR | 患者 ID (来自 clinic-service) | IDX |
| clinic_id | VARCHAR | 诊所 ID | IDX |
| doctor_id | VARCHAR | 医生 ID | IDX |
| department_id | VARCHAR | 科室 ID | IDX |
| schedule_id | VARCHAR | 排班 ID | FK, IDX |
| appointment_date | DATE | 预约日期 | IDX |
| time_slot_id | VARCHAR | 时段 ID (复用 schedule 的 time_slot) | IDX |
| status | VARCHAR | 预约状态 | IDX |
| visit_type | VARCHAR | 就诊类型 (初诊/复诊/急诊) | - |
| chief_complaint | TEXT | 主诉 | - |
| symptoms | TEXT | 症状描述 | - |
| check_in_time | TIMESTAMP | 签到时间 | - |
| actual_start_time | TIMESTAMP | 实际开始就诊时间 | - |
| actual_end_time | TIMESTAMP | 实际结束就诊时间 | - |
| cancel_reason | TEXT | 取消原因 | - |
| no_show_reason | TEXT | 爽约原因 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | - |

**索引策略**:
- `(tenant_id, status, appointment_date)` - 查询当日预约
- `(tenant_id, patient_id, created_at DESC)` - 查询患者历史预约
- `(tenant_id, doctor_id, appointment_date, status)` - 查询医生排班预约情况

#### schedules (排班表，参考 clinic-service，这里记录预约关联)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| doctor_id | VARCHAR | 医生 ID | IDX |
| department_id | VARCHAR | 科室 ID | IDX |
| work_date | DATE | 工作日期 | IDX |
| shift_type | VARCHAR | 班次类型 (上午/下午/夜班) | - |
| capacity | INT | 总容量 | - |
| booked_count | INT | 已预约数 | - |
| status | VARCHAR | 状态 (可用/已满/已取消) | IDX |

#### outbox_events (Outbox 模式)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | IDX |
| aggregate_type | VARCHAR | 聚合类型 (appointment) | IDX |
| aggregate_id | VARCHAR | 聚合 ID | IDX |
| event_type | VARCHAR | 事件类型 | - |
| event_data | JSONB | 事件数据 | - |
| published | BOOLEAN | 是否已发布 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

### 2.3 状态机设计

```
                    ┌─────────────┐
                    │   Pending   │ (已预约)
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Cancelled │    │  Checked  │    │ No-Show  │
    └──────────┘    └─────┬────┘    └──────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ In-Progress │ (就诊中)
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Completed   │ (已完成)
                    └─────────────┘
```

**状态说明**:
- `pending`: 已预约，等待就诊
- `checked_in`: 已签到，在队列中
- `in_progress`: 正在就诊
- `completed`: 就诊完成
- `cancelled`: 已取消（患者主动取消或诊所取消）
- `no_show`: 爽约（未在规定时间前签到）

---

## 3. API 接口设计

### 3.1 预约相关

#### POST /api/v1/appointments

创建预约

**请求**:
```json
{
  "doctor_id": "doctor_123",
  "schedule_id": "schedule_456",
  "appointment_date": "2026-04-15",
  "time_slot_id": "slot_001",
  "visit_type": "初诊",
  "chief_complaint": "牙痛持续3天",
  "symptoms": "右下颌磨牙区域疼痛"
}
```

**响应**:
```json
{
  "id": "apt_abc123",
  "status": "pending",
  "appointment_date": "2026-04-15",
  "time_slot": "09:00-09:30",
  "estimated_wait_time": 5,
  "check_in_window": {
    "start": "2026-04-15T08:30:00Z",
    "end": "2026-04-15T09:15:00Z"
  }
}
```

#### GET /api/v1/appointments/:id

获取预约详情

#### GET /api/v1/appointments

查询预约列表（支持分页、筛选）

**查询参数**:
- `status`: 预约状态
- `appointment_date`: 预约日期
- `patient_id`: 患者 ID
- `doctor_id`: 医生 ID

#### PUT /api/v1/appointments/:id/cancel

取消预约

**请求**:
```json
{
  "cancel_reason": "临时有事改期"
}
```

### 3.2 签到相关

#### POST /api/v1/appointments/:id/check-in

签到（支持患者扫码或前台代签）

**请求**:
```json
{
  "check_in_type": "self", // self 或 admin
  "location": "前台" // 可选，记录签到地点
}
```

**响应**:
```json
{
  "id": "apt_abc123",
  "status": "checked_in",
  "check_in_time": "2026-04-15T09:02:00Z",
  "queue_position": 3,
  "estimated_wait_minutes": 10
}
```

### 3.3 状态更新（BFF/内部）

#### PUT /api/v1/appointments/:id/status

更新预约状态（用于内部状态流转）

**请求**:
```json
{
  "status": "in_progress",
  "actual_start_time": "2026-04-15T09:12:00Z"
}
```

### 3.4 排班查询

#### GET /api/v1/schedules/available

查询可用号源

**查询参数**:
- `doctor_id`: 医生 ID
- `department_id`: 科室 ID
- `date_from`: 起始日期
- `date_to`: 结束日期

**响应**:
```json
{
  "schedules": [
    {
      "id": "schedule_456",
      "doctor_id": "doctor_123",
      "work_date": "2026-04-15",
      "shift_type": "上午",
      "time_slots": [
        {"id": "slot_001", "time": "09:00-09:30", "available": true, "capacity": 5, "booked": 2},
        {"id": "slot_002", "time": "09:30-10:00", "available": false, "capacity": 5, "booked": 5}
      ]
    }
  ]
}
```

---

## 4. 事件设计 (Outbox 模式)

### 4.1 领域事件

| 事件类型 | 触发时机 | 消费方 | 说明 |
|---------|---------|--------|------|
| `appointment.created` | 预约创建成功 | push-service, clinic-service | 发送预约确认通知 |
| `appointment.cancelled` | 预约取消 | push-service, payment-service | 发送取消通知，触发退款 |
| `appointment.checked_in` | 患者签到 | push-service, clinic-service | 更新排队状态 |
| `appointment.started` | 开始就诊 | emr-service, push-service | 触发病历创建通知 |
| `appointment.completed` | 就诊完成 | payment-service, emr-service | 触发费用计算 |
| `appointment.no_show` | 爽约 | push-service, clinic-service | 发送爽约提醒 |

### 4.2 事件数据结构

```json
{
  "event_type": "appointment.created",
  "aggregate_id": "apt_abc123",
  "tenant_id": "tenant_999",
  "data": {
    "appointment_id": "apt_abc123",
    "patient_id": "patient_123",
    "doctor_id": "doctor_123",
    "appointment_date": "2026-04-15",
    "time_slot": "09:00-09:30"
  },
  "occurred_at": "2026-04-15T09:00:00Z"
}
```

---

## 5. 防超卖设计

### 5.1 Redis 预扣 + 数据库事务确认

```
1. 预约请求到达
   ↓
2. Redis Lua 原子扣减号源
   └─ 成功: 返回 ticket_id
   └─ 失败: 返回号源已满
   ↓
3. 创建预约（数据库事务）
   └─ 成功: 发布事件，确认扣减
   └─ 失败: Redis 回滚，释放号源
   ↓
4. Outbox Relay 发布事件
```

### 5.2 Redis 数据结构

```
# 号源库存
booking:inventory:{tenant_id}:{schedule_id}:{time_slot_id} = count

# 预约凭证 (TTL 15 分钟)
booking:ticket:{ticket_id} = {patient_id, schedule_id, time_slot_id}
```

### 5.3 Lua 脚本

```lua
-- 扣减号源
local key = KEYS[1]
local ticket_id = ARGV[1]
local patient_id = ARGV[2]

local count = redis.call('GET', key)
if not count or tonumber(count) <= 0 then
    return {0, nil}  -- 失败
end

redis.call('DECR', key)
redis.call('SETEX', 'booking:ticket:' .. ticket_id, 900, patient_id)
return {1, ticket_id}  -- 成功
```

---

## 6. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `clinic-service` | gRPC | 查询医生、科室、排班信息 |
| `payment-service` | 事件消费 | 预约取消时触发退款 |
| `push-service` | gRPC | 推送预约、签到、叫号通知 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `Redis` | 直接连接 | 号源库存、缓存 |
| `NATS JetStream` | Outbox | 发布领域事件 |

---

## 7. 安全与合规

### 7.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 创建预约 | patient | `booking:create` |
| 取消预约 | patient, admin | `booking:cancel` |
| 签到 | patient, admin | `booking:check_in` |
| 更新状态 | doctor, admin | `booking:update_status` |
| 查看排班 | patient, doctor, admin | `booking:read_schedule` |

### 7.2 审计日志

记录以下关键操作：
- 预约创建/取消
- 签到/爽约
- 状态变更
- 操作人、时间、IP

---

## 8. 性能指标

| 指标 | 目标值 |
|------|--------|
| 创建预约响应时间 | < 200ms |
| 签到响应时间 | < 100ms |
| 排班查询响应时间 | < 300ms |
| 并发预约处理能力 | > 1000 TPS |
| 号源扣减准确性 | 100% |

---

## 9. 待讨论事项

1. **爽约时间窗口**: 超过预约时间多久算爽约？（建议：超过 15 分钟）
2. **取消预约退费规则**: 提前多久取消可全额退款？
3. **叫号顺序**: 按预约时间 vs 按签到顺序？
4. **跨天预约**: 是否支持跨时段预约？
5. **加号处理**: 医生主动加号的流程如何设计？

---

@gemini @gpt52
以上是 `booking-service` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: Patient Journey 是否完整？签到动效、排队展示的 UX 是否需要补充？
- **@gpt52**: 防超卖设计是否足够健壮？Outbox 模式是否有遗漏？是否有安全风险？
