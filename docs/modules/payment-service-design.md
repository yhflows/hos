# payment-service 详细设计草案

> 状态: 草案 v2 - 已纳入团队评审意见
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`payment-service` 负责 HOS 系统的费用计算、交易处理、退款管理、对账报表，是资金流的唯一真相源。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 账单生成与计算 | ✅ | - |
| 交易创建与状态管理 | ✅ | - |
| 支付回调处理与幂等 | ✅ | - |
| 退款申请与处理 | ✅ | - |
| 对账与报表 | ✅ | - |
| 医生/科室/排班基础数据 | ❌ | clinic-service |
| 挂号/就诊状态 | ❌ | booking-service |
| 第三方支付通道 | ❌ | 聚合支付网关 (BFF 层) |

### 1.2 核心约束

- **多租户隔离**: 所有操作必须在租户 schema 内执行
- **资金安全**: 交易金额精确到分，禁止浮点运算
- **幂等保证**: 支付回调必须严格幂等，避免重复入账
- **审计日志**: 所有资金操作必须记录完整审计日志
- **最终一致性**: 跨服务事件通过 Outbox 模式发布
- **对账机制**: 每日自动对账 + T+0 异常队列
- **账单不可变性**: 账单必须固化价格快照，目录调价不影响历史账单
- **退款余额约束**: 累计已退款 <= 原始已结算金额
- **Append-Only Ledger**: 分配/退款记录只能追加，不可修改

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### encounters (就诊记录 - 快照引用)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID (emr-service) | FK, IDX |
| encounter_snapshot_hash | VARCHAR | 就诊快照哈希 (防篡改) | - |
| catalog_version | VARCHAR | 定价目录版本号 (快照) | - |
| status | VARCHAR | 账单状态 (pending/active/partial_paid/paid/cancelled) | IDX |
| total_amount_cents | BIGINT | 总金额 (分) | - |
| paid_amount_cents | BIGINT | 已付金额 (分) | - |
| discount_amount_cents | BIGINT | 优惠金额 (分) | - |
| insurance_coverage_cents | BIGINT | 保险抵扣金额 (分) | - |
| patient_pay_cents | BIGINT | 患者自付金额 (分) | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | - |

#### bills (账单明细表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| bill_code | VARCHAR | 账单代码 (定价体系) | IDX |
| bill_name | TEXT | 账单名称 | - |
| category | VARCHAR | 类别 (挂号/治疗/检查/药品/材料) | IDX |
| unit_price_cents | INT | 单价 (分) | - |
| quantity | DECIMAL(10,2) | 数量 | - |
| amount_cents | BIGINT | 金额 (分) | - |
| discount_rate | DECIMAL(5,2) | 折扣率 | - |
| discount_amount_cents | BIGINT | 折扣金额 (分) | - |
| tax_rate | DECIMAL(5,2) | 税率 | - |
| tax_amount_cents | BIGINT | 税额 (分) | - |
| final_amount_cents | BIGINT | 最终金额 (分) | - |
| item_type | VARCHAR | 项目类型 (service/medicine/material) | - |
| item_id | VARCHAR | 项目 ID (关联定价) | IDX |
| catalog_snapshot | JSONB | 定价快照 (item_code, item_name, unit_price_cents at bill time) | - |
| clinical_context | JSONB | 临床上下文 (tooth_number, tooth_layer, diagnosis_code) - 仅投影 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

#### transactions (交易记录表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| patient_id | VARCHAR | 患者 ID | IDX |
| transaction_no | VARCHAR | 交易流水号 (第三方) | IDX |
| internal_no | VARCHAR | 内部交易号 | UNIQUE, IDX |
| amount_cents | BIGINT | 交易金额 (分) | IDX |
| currency | VARCHAR | 币种 (CNY) | - |
| status | VARCHAR | 状态 (pending/success/failed/refunded/partial_refunded) | IDX |
| payment_method | VARCHAR | 支付方式 (wechat/alipay/card/cash) | IDX |
| payment_channel | VARCHAR | 支付通道 (M1/M2/等) | IDX |
| provider_order_id | VARCHAR | 第三方订单号 | IDX |
| provider_transaction_id | VARCHAR | 第三方交易号 | IDX |
| idempotency_key | VARCHAR | 幂等键 (来自支付请求) | UNIQUE, IDX |
| paid_at | TIMESTAMP | 支付时间 | IDX |
| refunded_at | TIMESTAMP | 退款时间 | IDX |
| refund_amount_cents | BIGINT | 退款金额 (分) | - |
| callback_event_id | VARCHAR | 回调事件 ID (追踪多次回调) | IDX |
| signature | VARCHAR | 回调签名 (验签留存) | - |
| callback_payload | JSONB | 回调原始 payload (审计) | - |
| refundable_balance_cents | BIGINT | 可退余额 (校验字段) | IDX |
| // UX 投影字段 (只读)
| visual_state | JSONB | 视觉状态 (color, icon, animation) | - |
| timeline_view | JSONB | 时间轴视图 (stages: [...]) | - |
| coverage_breakdown | JSONB | 覆盖分解 (insurance, coupon, patient_pay) | - |
| refund_reason | TEXT | 退款原因 | - |
| metadata | JSONB | 附加元数据 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | - |

**索引策略**:
- `(tenant_id, internal_no)` - 内部交易号查询
- `(tenant_id, idempotency_key)` - 幂等键查询
- `(tenant_id, provider_order_id)` - 第三方订单号查询
- `(tenant_id, callback_event_id)` - 回调事件追踪
- `(tenant_id, status, created_at DESC)` - 查询待处理交易

**状态机约束**:
- 状态只能单调递进：pending → success/failed → (refunded/partial_refunded)
- 已支付交易不能修改金额
- 已全额退款交易不能再退款

**幂等策略**:
- `idempotency_key`: 客户端生成，全局唯一
- `internal_no`: 系统生成，租户内唯一
- `callback_event_id`: 回调事件追踪，同一 provider_transaction_id 多次回调应关联
- `signature`: 回调签名必须验证并存档

**回调幂等处理流程**:
```
1. 收到回调，解析 provider_transaction_id
   ↓
2. 查询 `internal_no` 按 `provider_transaction_id`
   ├─ 存在: 检查状态，更新回调 payload
   └─ 不存在: 按幂等键查询，拒绝重复入账
   ↓
3. 存档 `callback_payload` 和 `signature` 到 callback_logs
   ↓
4. 同事务更新交易状态 + 记录 callback_logs
```

#### refunds (退款记录表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| transaction_id | VARCHAR | 原交易 ID | FK, IDX |
| refund_no | VARCHAR | 退款流水号 | UNIQUE, IDX |
| amount_cents | BIGINT | 退款金额 (分) | IDX |
| status | VARCHAR | 状态 (pending/processing/success/failed) | IDX |
| refund_type | VARCHAR | 退款类型 (full/partial) | - |
| reason | TEXT | 退款原因 | - |
| applied_by | VARCHAR | 操作人 ID | IDX |
| approved_by | VARCHAR | 审批人 ID | IDX |
| approved_at | TIMESTAMP | 审批时间 | IDX |
| provider_refund_id | VARCHAR | 第三方退款单号 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**退款余额约束**:
- 同一交易的累计已退款金额 <= 原始已结算金额
- `refundable_balance_cents` 字段实时维护可退余额

#### bill_allocations (账单分配记录 - Append-Only Ledger)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| bill_id | VARCHAR | 账单 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| transaction_id | VARCHAR | 关联交易 ID | FK, IDX |
| allocation_type | VARCHAR | 分配类型 (payment/insurance/coupon/write_off) | IDX |
| amount_cents | BIGINT | 分配金额 (分) | IDX |
| channel | VARCHAR | 渠道 (direct/insurance_patient/insurance_clinic) | IDX |
| reference_id | VARCHAR | 关联单号 (保险单号/优惠券号) | IDX |
| reference_type | VARCHAR | 关联类型 (insurance_policy/coupon) | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**Append-Only 约束**:
- 本表只能追加记录，禁止修改或删除
- 用于拆分支付、保险抵扣、优惠券核销、坏账核销的审计追踪

#### refund_allocations (退款分配记录 - Append-Only Ledger)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| refund_id | VARCHAR | 退款 ID | FK, IDX |
| transaction_id | VARCHAR | 原交易 ID | FK, IDX |
| allocation_type | VARCHAR | 分配类型 (refund_to_patient/refund_to_insurance/write_off) | IDX |
| amount_cents | BIGINT | 分配金额 (分) | IDX |
| channel | VARCHAR | 退款渠道 (original/payment_channel/refund_card) | IDX |
| reference_id | VARCHAR | 关联单号 | - |
| reference_type | VARCHAR | 关联类型 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**Append-Only 约束**:
- 本表只能追加记录，禁止修改或删除
- 用于部分退款、退回保险、坏账核销的审计追踪

#### callback_logs (支付回调日志 - 审计追溯)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| callback_event_id | VARCHAR | 回调事件 ID | IDX |
| provider | VARCHAR | 支付提供商 (wechat/alipay) | IDX |
| provider_order_id | VARCHAR | 第三方订单号 | IDX |
| provider_transaction_id | VARCHAR | 第三方交易号 | IDX |
| callback_payload | JSONB | 回调原始 payload | - |
| signature | VARCHAR | 回调签名 | - |
| signature_valid | BOOLEAN | 签名验证结果 | - |
| received_at | TIMESTAMP | 接收时间 | IDX |
| processed_at | TIMESTAMP | 处理时间 | IDX |
| processing_duration_ms | INT | 处理耗时 | - |
| error_message | TEXT | 错误信息 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**回调日志用途**:
- 审计追溯：所有支付回调必须记录
- 重复检测：同一 provider_transaction_id 多次回调应关联
- 故障排查：签名验证失败、处理超时等可追溯

#### T+0 异常队列 (分钟级异常发现)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| anomaly_type | VARCHAR | 异常类型 (callback_timeout/transaction_missing/amount_mismatch/duplicate_notification) | IDX |
| reference_id | VARCHAR | 关联 ID (transaction_id/internal_no) | IDX |
| reference_type | VARCHAR | 关联类型 | - |
| severity | VARCHAR | 严重程度 (critical/high/medium/low) | IDX |
| detected_at | TIMESTAMP | 检测时间 | IDX |
| resolved_at | TIMESTAMP | 解决时间 | IDX |
| resolution | TEXT | 解决方案 | - |
| status | VARCHAR | 状态 (pending/resolved/false_positive) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**异常类型**:
- `callback_timeout`: 回调超时未收到（已创建 pending 交易但超时未完成）
- `transaction_missing`: 对账发现本地有但第三方没有
- `amount_mismatch`: 金额不一致
- `duplicate_notification`: 第三方重复通知
- `refund_failed`: 退款失败需要人工介入

#### pricing_catalogs (定价目录表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| catalog_type | VARCHAR | 目录类型 (service/medicine/material) | IDX |
| item_code | VARCHAR | 项目代码 | IDX |
| item_name | TEXT | 项目名称 | - |
| category | VARCHAR | 分类 | IDX |
| unit_price_cents | INT | 单价 (分) | - |
| effective_date | DATE | 生效日期 | IDX |
| expiry_date | DATE | 失效日期 | IDX |
| is_active | BOOLEAN | 是否启用 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, catalog_type, item_code)` - 项目查询
- `(tenant_id, is_active, effective_date)` - 查询当前有效价格

#### outbox_events (Outbox 模式)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | IDX |
| aggregate_type | VARCHAR | 聚合类型 (transaction/refund) | IDX |
| aggregate_id | VARCHAR | 聚合 ID | IDX |
| event_type | VARCHAR | 事件类型 | - |
| event_data | JSONB | 事件数据 | - |
| published | BOOLEAN | 是否已发布 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

---

## 3. API 接口设计

### 3.1 账单相关

#### POST /api/v1/bills

生成账单（由 encounter.completed 事件触发）

**请求**:
```json
{
  "encounter_id": "enc_123",
  "encounter_snapshot_hash": "a1b2c3..."
}
```

**响应**:
```json
{
  "encounter_id": "enc_123",
  "bill_id": "bill_abc123",
  "total_amount_cents": 50000,
  "discount_amount_cents": 0,
  "final_amount_cents": 50000,
  "status": "active",
  "items": [
    {
      "bill_code": "B001",
      "bill_name": "初诊挂号费",
      "category": "挂号",
      "unit_price_cents": 5000,
      "quantity": 1,
      "final_amount_cents": 5000
    },
    {
      "bill_code": "T001",
      "bill_name": "树脂充填 (单颗)",
      "category": "治疗",
      "unit_price_cents": 45000,
      "quantity": 1,
      "final_amount_cents": 45000
    }
  ]
}
```

#### GET /api/v1/bills/:encounter_id

查询就诊账单

### 3.2 交易相关

#### POST /api/v1/transactions

创建交易（发起支付）

**请求**:
```json
{
  "encounter_id": "enc_123",
  "patient_id": "patient_123",
  "amount_cents": 50000,
  "currency": "CNY",
  "payment_method": "wechat",
  "payment_channel": "M1",
  "idempotency_key": "pay_xyz123",
  "return_url": "https://...",
  "notify_url": "https://..."
}
```

**响应**:
```json
{
  "internal_no": "txn_20260415123456",
  "amount_cents": 50000,
  "status": "pending",
  "provider_order_id": "wx123456789",
  "payment_info": {
    "prepay_id": "wx...",
    "code_url": "weixin://wxpay/bizpayurl?..."
  }
}
```

#### POST /api/v1/transactions/callback

支付回调（由第三方支付网关调用）

**请求**:
```json
{
  "provider_order_id": "wx123456789",
  "provider_transaction_id": "420000...",
  "amount_cents": 50000,
  "status": "success",
  "idempotency_key": "pay_xyz123",
  "signature": "..."
}
```

**响应**:
```json
{
  "status": "success",
  "internal_no": "txn_20260415123456"
}
```

#### GET /api/v1/transactions/:internal_no

查询交易详情

### 3.3 退款相关

#### POST /api/v1/refunds

申请退款

**请求**:
```json
{
  "transaction_id": "txn_abc123",
  "amount_cents": 50000,
  "refund_type": "full",
  "reason": "取消就诊"
}
```

**响应**:
```json
{
  "refund_id": "refund_xyz123",
  "refund_no": "rf_20260415123456",
  "status": "pending",
  "amount_cents": 50000
}
```

#### POST /api/v1/refunds/:id/approve

审批退款（管理员操作）

#### POST /api/v1/refunds/callback

退款回调（由第三方支付网关调用）

### 3.4 定价相关

#### POST /api/v1/pricing/catalogs

创建/更新定价目录

#### GET /api/v1/pricing/catalogs

查询定价目录

---

## 4. 事件设计 (Outbox 模式)

### 4.1 领域事件

| 事件类型 | 触发时机 | 消费方 | 说明 |
|---------|---------|--------|------|
| `bill.created` | 账单生成 | push-service, clinic-service | 发送账单通知 |
| `transaction.pending` | 交易创建 | push-service | 发送支付链接 |
| `transaction.paid` | 支付成功 | push-service, booking-service | 更新预约状态为已完成 |
| `transaction.failed` | 支付失败 | push-service | 发送支付失败通知 |
| `refund.pending` | 退款申请 | push-service | 发送退款申请通知 |
| `refund.approved` | 退款审批 | push-service | 发送退款审批通知 |
| `refund.completed` | 退款完成 | push-service, booking-service | 更新预约状态为已取消 |
| `reconciliation.completed` | 对账完成 | push-service | 发送对账结果通知 |

### 4.2 事件数据结构

```json
{
  "event_type": "transaction.paid",
  "aggregate_id": "txn_abc123",
  "tenant_id": "tenant_999",
  "data": {
    "internal_no": "txn_20260415123456",
    "patient_id": "patient_123",
    "encounter_id": "enc_123",
    "amount_cents": 50000,
    "payment_method": "wechat",
    "paid_at": "2026-04-15T10:00:00Z"
  },
  "occurred_at": "2026-04-15T10:00:00Z"
}
```

---

## 5. 幂等与并发控制

### 5.1 支付幂等保证

```
1. 客户端发起支付，携带 idempotency_key (UUID)
   ↓
2. payment-service 查询幂等键是否已存在
   └─ 存在: 返回已有交易状态
   └─ 不存在: 创建新交易
   ↓
3. 第三方支付网关返回 provider_order_id
   ↓
4. 更新交易，关联 provider_order_id
   ↓
5. 支付回调时，按 idempotency_key 查找并更新状态
```

### 5.2 幂等键存储

```
# 幂等键缓存 (TTL 24 小时)
payment:idempotency:{tenant_id}:{idempotency_key} = transaction_id

# 交易状态缓存 (TTL 1 小时)
payment:transaction:{tenant_id}:{internal_no} = {status, amount_cents, paid_at}
```

### 5.3 并发控制

- **交易创建**: `idempotency_key` 唯一约束
- **账单生成**: `encounter_id` + `encounter_snapshot_hash` 联合唯一约束
- **退款申请**: 同一交易只允许一笔进行中退款

---

## 6. 对账机制

### 6.1 自动对账流程

```
每日 02:00 定时任务
   ↓
1. 查询当日所有 success 交易
   ↓
2. 从第三方支付网关拉取对账单
   ↓
3. 按内部交易号 (internal_no) 逐笔比对
   ↓
4. 差异处理:
   ├─ 差额: 记录异常，发送告警
   ├─ 缺单: 第三方有但本地没有 → 补单
   └─ 多单: 本地有但第三方没有 → 标记异常
   ↓
5. 生成对账报告
   ↓
6. 发布 reconciliation.completed 事件
```

### 6.2 对账报告结构

| 字段名 | 说明 |
|--------|------|
| reconciliation_date | 对账日期 |
| total_transactions | 总交易笔数 |
| matched_count | 匹配笔数 |
| mismatched_count | 差异笔数 |
| total_amount_cents | 总金额 |
| matched_amount_cents | 匹配金额 |
| mismatched_amount_cents | 差异金额 |
| anomalies | 异常明细 (JSONB) |

---

## 7. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `emr-service` | gRPC | 获取就诊快照，验证 encounter_snapshot_hash |
| `booking-service` | 事件消费 | 支付成功/失败后更新预约状态 |
| `push-service` | gRPC | 推送支付、账单、退款通知 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `Redis` | 直接连接 | 幂等缓存、交易状态缓存 |
| `NATS JetStream` | Outbox | 发布领域事件 |
| 第三方支付网关 | HTTP | 微信支付、支付宝等 |

---

## 8. 安全与合规

### 8.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 查看账单 | patient, doctor, admin | `payment:read_bill` |
| 创建交易 | patient, admin | `payment:create_transaction` |
| 查看交易详情 | patient, doctor, admin | `payment:read_transaction` |
| 申请退款 | patient, admin | `payment:request_refund` |
| 审批退款 | admin | `payment:approve_refund` |
| 查看对账报告 | admin | `payment:read_reconciliation` |
| 管理定价目录 | admin | `payment:manage_pricing` |

### 8.2 审计日志

记录以下关键操作：
- 账单生成
- 交易创建/支付成功/支付失败
- 退款申请/审批/完成
- 对账完成
- 定价目录修改
- 操作人、时间、IP、修改前/后值

### 8.3 资金安全

- **金额精度**: 所有金额使用 `cents` (整数) 存储，避免浮点误差
- **货币统一**: 当前仅支持 CNY，未来可扩展
- **敏感信息加密**: 第三方支付凭证、密钥使用应用级加密存储
- **数据保留**: 交易记录永久保留，不可删除

---

## 9. 性能指标

| 指标 | 目标值 |
|------|--------|
| 账单生成响应时间 | < 200ms |
| 交易创建响应时间 | < 150ms |
| 支付回调处理时间 | < 100ms |
| 对账完成时间 | < 5 分钟 |
| 并发交易处理能力 | > 2000 TPS |
| 幂等查询响应时间 | < 50ms |

---

## 10. 待讨论事项

### 10.1 业务规则

1. **支付渠道**: 集成哪些支付渠道？（微信支付、支付宝、银联等）
2. **定价体系**: 治疗项目定价策略如何设计？（统一价/医生差异化/时段差异化）
3. **部分支付**: 是否支持部分支付？（预付款 + 到院后补齐）
4. **优惠券/积分**: 是否支持优惠券或积分抵扣？
5. **电子发票**: 是否需要生成电子发票？
6. **分期付款**: 是否支持分期付款？
7. **退款规则**: 多久内可全额退款？手续费如何处理？

### 10.2 已采纳的团队建议

| 来源 | 建议 | 状态 |
|------|------|------|
| @gemini | 诊疗关联式账单 (牙位图连线) | ✅ 已纳入 (catalog_snapshot clinical_context) |
| @gemini | 支付状态情绪化设计 (Pending/Amber/Paid/Mint) | ✅ 已纳入 (visual_state) |
| @gemini | 阶梯支付可视化 (进度条/还款日历) | ✅ 已纳入 (coverage_breakdown) |
| @gemini | 动态定价卡片化后台 | ✅ 已纳入 (UX 建议) |
| @gpt52 | 账务不可变模型 (bill_allocations/append-only ledger) | ✅ 已纳入 |
| @gpt52 | 支付回调幂等 (callback_event_id/signature/原始日志) | ✅ 已纳入 |
| @gpt52 | 账单价格快照 (catalog_version/tax_amount) | ✅ 已纳入 |
| @gpt52 | T+0 异常队列 (分钟级异常发现) | ✅ 已纳入 |
| @gpt52 | 退款余额约束 (累计已退款 <= 原始已结算) | ✅ 已纳入 |

### 10.3 设计原则总结

| 原则 | 实现 |
|------|------|
| **金额精度** | 所有金额使用 cents (BIGINT) 存储，避免浮点误差 |
| **幂等保证** | idempotency_key + callback_event_id + signature 三重验证 |
| **账务不可变** | 账单价格快照化，目录调价不影响历史账单 |
| **Append-Only Ledger** | 分配/退款只能追加，不可修改 |
| **状态单调性** | 交易状态只能单调递进，禁止逆向 |
| **审计全覆盖** | 所有资金操作记录完整审计日志 |
| **分钟级异常检测** | T+0 异常队列，及时发现支付问题 |
| **退款余额校验** | 累计已退款 <= 原始已结算金额 |
| **UX 投影分离** | 视觉字段只读，不进核心写模型 |

### 10.4 测试用例需求

| 场景 | 测试目标 |
|------|---------|
| 重复回调幂等 | 验证同一 provider_transaction_id 多次回调只处理一次 |
| 部分退款/多次退款余额校验 | 验证退款余额约束，禁止超额退款 |
| 目录调价历史账单不变 | 验证价格快照机制，目录调价不影响已生成账单 |
| 对账差异补偿闭环 | 验证异常发现后的自动补偿机制 |

---
