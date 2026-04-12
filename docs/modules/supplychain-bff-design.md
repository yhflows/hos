# supplychain-bff 详细设计草案

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`supplychain-bff` 是 HOS 系统中 Odoo 的防腐层集成服务，负责租户到 Odoo Company 映射、RPC 适配、错误翻译、限流与重试。MVP 阶段不做在线强依赖，采用异步对账模式。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 租户 → Odoo Company 映射 | ✅ | - |
| Odoo RPC 适配 | ✅ | - |
| 错误翻译与重试 | ✅ | - |
| 异步库存对账 | ✅ | - |
| 租户开通/退租 Odoo 编排 | ✅ | - |
| 消耗记录推送 | ✅ | - |
| 实时库存锁定 | ❌ | MVP 暂缓 |
| 在线支付强依赖 Odoo | ❌ | 核心链路独立 |
| 采购订单管理 | ❌ | 由 Odoo 直接管理 |

### 1.2 核心约束

- **异步优先**: 核心支付链路不等待 Odoo 在线响应
- **防腐层隔离**: 禁止业务服务直接操作 Odoo
- **限流保护**: 保护 Odoo 免受过载
- **幂等保证**: 对账和推送操作必须幂等
- **审计日志**: 所有 Odoo 操作必须记录

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### tenant_odoo_mappings (租户 Odoo 映射表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, UNIQUE, IDX |
| odoo_company_id | INT | Odoo Company ID | IDX |
| odoo_db_name | VARCHAR | Odoo 数据库名 | - |
| odoo_api_endpoint | VARCHAR | Odoo API 端点 | - |
| odoo_api_key | VARCHAR | Odoo API 密钥（加密） | - |
| mapping_status | VARCHAR | 映射状态 (pending/active/inactive) | IDX |
| mapped_at | TIMESTAMP | 映射时间 | IDX |
| unmap_reason | VARCHAR | 解绑原因 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id)` - 唯一约束
- `(odoo_company_id)` - Odoo Company 查询

#### product_mappings (产品映射表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| internal_product_id | VARCHAR | 内部产品 ID | UNIQUE, IDX |
| odoo_product_id | INT | Odoo 产品 ID | IDX |
| odoo_product_code | VARCHAR | Odoo 产品编码 | - |
| product_name | VARCHAR | 产品名称（快照） | - |
| sync_status | VARCHAR | 同步状态 (synced/pending/fail) | IDX |
| last_synced_at | TIMESTAMP | 最后同步时间 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, internal_product_id)` - 唯一约束
- `(tenant_id, odoo_product_id)` - Odoo 产品查询

#### inventory_sync_logs (库存同步日志表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊 ID | FK, IDX |
| treatment_id | VARCHAR | 治疗 ID | FK, IDX |
| product_id | VARCHAR | 产品 ID | IDX |
| quantity | INT | 消耗数量 | - |
| odoo_company_id | INT | Odoo Company ID | IDX |
| odoo_product_id | INT | Odoo 产品 ID | IDX |
| sync_status | VARCHAR | 同步状态 (pending/success/failed) | IDX |
| retry_count | INT | 重试次数 | IDX |
| error_message | TEXT | 错误信息 | - |
| odoo_transaction_id | VARCHAR | Odoo 事务 ID | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| synced_at | TIMESTAMP | 同步完成时间 | IDX |

**索引策略**:
- `(tenant_id, sync_status, retry_count)` - 待重试查询
- `(tenant_id, encounter_id)` - 就诊关联查询
- `(tenant_id, created_at DESC)` - 最新同步记录

#### reconciliation_jobs (对账任务表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| job_type | VARCHAR | 任务类型 (inventory/billing) | IDX |
| job_status | VARCHAR | 任务状态 (pending/running/completed/failed) | IDX |
| start_time | TIMESTAMP | 开始时间 | IDX |
| end_time | TIMESTAMP | 结束时间 | - |
| processed_count | INT | 处理记录数 | - |
| failed_count | INT | 失败记录数 | - |
| error_summary | JSONB | 错误摘要 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, job_status, created_at DESC)` - 任务查询

#### odoo_rpc_logs (Odoo RPC 调用日志表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| odoo_company_id | INT | Odoo Company ID | IDX |
| rpc_method | VARCHAR | RPC 方法 | IDX |
| request_payload | JSONB | 请求负载 | - |
| response_payload | JSONB | 响应负载 | - |
| status | VARCHAR | 状态 (success/failed/timeout) | IDX |
| duration_ms | INT | 耗时（毫秒） | - |
| error_code | VARCHAR | 错误码 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, status, created_at DESC)` - 失败查询
- `(tenant_id, odoo_company_id, created_at DESC)` - Company 调用历史

---

## 3. API 接口设计

### 3.1 租户映射管理

#### POST /api/v1/tenants/:tenant_id/odoo/map

映射租户到 Odoo Company

**请求体**:
```json
{
  "odoo_company_id": 1,
  "odoo_db_name": "hos_clinic_01",
  "odoo_api_endpoint": "https://odoo.example.com/xmlrpc",
  "odoo_api_key": "encrypted_key_here"
}
```

**响应**:
```json
{
  "tenant_id": "tenant_999",
  "odoo_company_id": 1,
  "mapping_status": "active",
  "mapped_at": "2026-04-12T10:00:00Z"
}
```

#### DELETE /api/v1/tenants/:tenant_id/odoo/unmap

解绑租户映射

**请求体**:
```json
{
  "reason": "租户到期退租"
}
```

### 3.2 产品映射管理

#### POST /api/v1/products/sync

同步产品到 Odoo

**请求体**:
```json
{
  "product_id": "prod_001",
  "product_name": "种植体 A 型",
  "product_code": "IMPLANT-A"
}
```

#### GET /api/v1/products/mappings

查询产品映射

**查询参数**:
- `sync_status`: 同步状态过滤

### 3.3 库存消耗推送

#### POST /api/v1/inventory/consume

推送库存消耗记录

**请求体**:
```json
{
  "encounter_id": "enc_001",
  "consumptions": [
    {
      "product_id": "prod_001",
      "quantity": 1
    },
    {
      "product_id": "prod_002",
      "quantity": 2
    }
  ]
}
```

**响应**:
```json
{
  "job_id": "job_abc123",
  "status": "pending",
  "message": "库存消耗记录已加入对账队列"
}
```

### 3.4 对账任务管理

#### POST /api/v1/reconciliation/trigger

触发对账任务

**请求体**:
```json
{
  "job_type": "inventory",
  "date_range": {
    "from": "2026-04-10",
    "to": "2026-04-12"
  }
}
```

#### GET /api/v1/reconciliation/jobs

查询对账任务

**查询参数**:
- `job_type`: 任务类型
- `status`: 状态过滤
- `page`: 页码

---

## 4. gRPC 服务定义

### 4.1 Supplychain Service

```protobuf
service SupplychainService {
  // 租户映射
  rpc MapTenant(MapTenantRequest) returns (MapTenantResponse);
  rpc UnmapTenant(UnmapTenantRequest) returns (google.protobuf.Empty);
  rpc GetOdooCompany(GetOdooCompanyRequest) returns (OdooCompany);

  // 产品映射
  rpc SyncProduct(SyncProductRequest) returns (SyncProductResponse);
  rpc GetProductMapping(GetProductMappingRequest) returns (ProductMapping);

  // 库存消耗
  rpc PushInventoryConsumption(PushInventoryConsumptionRequest) returns (PushInventoryConsumptionResponse);

  // 对账
  rpc TriggerReconciliation(TriggerReconciliationRequest) returns (TriggerReconciliationResponse);
  rpc GetReconciliationStatus(GetReconciliationStatusRequest) returns (ReconciliationStatus);
}
```

---

## 5. Odoo RPC 适配

### 5.1 RPC 方法映射

| 业务操作 | Odoo RPC 方法 | 说明 |
|---------|--------------|------|
| 产品创建/更新 | product.product/create_or_update | 同步产品信息 |
| 库存扣减 | stock.quant.write | 异步扣减库存 |
| 库存查询 | stock.quant.search_read | 对账时查询库存 |
| 移动订单 | stock.move/create | 创建库存移动记录 |

### 5.2 错误翻译

| Odoo 错误码 | 业务含义 | 处理策略 |
|------------|---------|---------|
| AccessError | 权限不足 | 检查 API 密钥和 Company 映射 |
| ValidationError | 数据验证失败 | 检查产品映射完整性 |
| UserError | 用户错误 | 返回给用户修正 |
| OperationalError | 操作错误 | 重试后告警 |
| ConnectionError | 连接失败 | 标记为失败，放入重试队列 |

### 5.3 限流策略

| 资源 | 限流阈值 | 目的 |
|------|---------|------|
| 单租户 RPC 调用 | 100 req/min | 保护 Odoo |
| 单 Odoo Company 调用 | 500 req/min | 保护 Company 实例 |
| 对账任务 | 1 task/5min | 避免重叠 |

---

## 6. 异步对账流程

### 6.1 库存消耗对账

```
1. emr-service 完成就诊，创建治疗记录
   ↓
2. 发布 treatment.created 事件
   ↓
3. supplychain-bff 订阅事件，创建 inventory_sync_logs
   ↓
4. 异步推送库存消耗到 Odoo
   ├─ 成功: 标记 sync_status = success
   └─ 失败: 标记 sync_status = failed，加入重试队列
   ↓
5. 定时任务扫描失败记录，重试 3 次
   ↓
6. 3 次重试仍失败: 告警
```

### 6.2 定时对账任务

```
每日 02:00 执行全量对账
   ↓
1. 查询昨日所有已完成就诊
   ↓
2. 汇总库存消耗记录
   ↓
3. 对比 Odoo 库存数据
   ↓
4. 生成差异报告
   ↓
5. 推送差异到管理员
```

---

## 7. Outbox 事件订阅

### 7.1 事件类型

| 事件类型 | 发布服务 | 处理逻辑 |
|---------|---------|---------|
| `treatment.created` | emr-service | 推送库存消耗到 Odoo |
| `encounter.completed` | emr-service | 创建 Odoo 销售订单 |
| `payment.completed` | payment-service | 更新 Odoo 财务记录 |

---

## 8. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `emr-service` | NATS | 订阅治疗完成事件 |
| `payment-service` | NATS | 订阅支付完成事件 |
| `Odoo 17+` | XML-RPC | 库存、财务同步 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `NATS JetStream` | 直接连接 | 订阅业务事件 |

---

## 9. 安全与合规

### 9.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 映射/解绑租户 | admin | `supplychain:manage_tenant_mapping` |
| 同步产品 | admin | `supplychain:sync_product` |
| 触发对账 | admin | `supplychain:trigger_reconciliation` |
| 查看对账报告 | admin | `supplychain:read_report` |

### 9.2 安全措施

- **API 密钥加密**: Odoo API 密钥使用 AES-256 加密存储
- **审计日志**: 所有 Odoo RPC 调用记录审计日志
- **重试限流**: 防止重试导致 Odoo 过载
- **数据脱敏**: 日志中脱敏敏感信息

---

## 10. 性能指标

| 指标 | 目标值 |
|------|--------|
| 租户映射响应时间 | < 200ms |
| 产品同步响应时间 | < 500ms |
| 库存消耗推送延迟 | < 5 分钟 |
| 对账任务完成时间 | < 30 分钟 |
| RPC 调用成功率 | > 99% |

---

## 11. 待讨论事项

1. **Odoo 版本**: 最终采用哪个版本的 Odoo？（草案: Odoo 17+）
2. **多仓库支持**: 是否需要支持同一租户多个仓库？
3. **回滚策略**: Odoo 同步失败时如何回滚业务操作？
4. **实时库存**: Phase 2 是否需要实时库存锁定？
5. **价格同步**: 是否需要将价格目录同步到 Odoo？

---

@gemini @gpt52
以上是 `supplychain-bff` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: 对账报告展示是否清晰？操作流程是否需要优化？
- **@gpt52**: RPC 适配是否完整？重试策略是否健壮？幂等性是否保证？是否有安全风险？
