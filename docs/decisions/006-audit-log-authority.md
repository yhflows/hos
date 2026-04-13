# ADR-006: 审计日志权威来源 (Audit Log Authority)

> 状态: 草案 - 待团队评审
> 创建日期: 2026-04-13
> 作者: @opus (布偶猫)

---

## 1. 状态 (Status)
草案 (Draft) - 待团队评审

## 2. 上下文 (Context)

医疗行业审计要求：
- 审计记录不可篡改
- 审计链条可追溯
- 证据来源唯一且可信

## 3. 决策结果 (Decision)

### 3.1 审计日志权威来源

**决策**: 后端审计日志是审计记录的唯一权威来源

**要求**:
1. 审计日志必须在后端服务层记录，禁止依赖前端上报
2. 审计日志记录包含完整信息：操作者、操作类型、前后 diff、时间戳、IP
3. 审计日志必须写入到不可变的 Append-Only 表结构
4. 审计日志禁止删除或修改（只能归档）

### 3.2 审计日志表结构

```sql
CREATE TABLE audit_logs (
  id ULID PRIMARY KEY,
  tenant_id VARCHAR NOT NULL,
  user_id VARCHAR NOT NULL,
  action VARCHAR NOT NULL,           -- 操作类型
  resource_type VARCHAR NOT NULL,      -- 资源类型
  resource_id VARCHAR,                 -- 资源 ID
  old_value JSONB,                    -- 操作前值
  new_value JSONB,                    -- 操作后值
  ip_address VARCHAR,                  -- 操作者 IP
  user_agent TEXT,                     -- User Agent
  created_at TIMESTAMP NOT NULL,
  signature VARCHAR,                    -- 数字签名（可选，用于高安全场景）
  INDEX idx_tenant_user (tenant_id, user_id, created_at DESC),
  INDEX idx_resource (tenant_id, resource_type, resource_id, created_at DESC)
);
```

**关键约束**:
- `CHECK(old_value IS NOT NULL OR new_value IS NOT NULL)` - 至少记录一个状态
- 禁止 UPDATE/DELETE 操作（只允许 INSERT）

### 3.3 前端快照的作用

**决策**: 前端快照仅作为辅助材料，不能作为审计证据

**用途**:
1. UI 层操作回退时的本地恢复
2. 争议排查时的上下文参考
3. 性能优化（避免频繁查询后端）

**限制**:
1. 前端快照不作为审计依据
2. 前端快照可随时被清除
3. 前端快照不能篡改后端审计记录

### 3.4 审计日志查询接口

**要求**: 查询接口必须带权限验证

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 查看审计日志 | admin | `audit:read` |
| 导出审计日志 | admin | `audit:export` |

**审计日志查询**:
```http
GET /api/v1/audit/logs?
  tenant_id=tenant_999&
  user_id=user_123&
  resource_type=appointment&
  date_from=2026-04-01&
  date_to=2026-04-13
```

## 4. 后果 (Consequences)

- 审计记录有单一真相源
- 即使前端被篡改，后端审计记录仍然可信
- 前端快照功能受限，仅作辅助用途
- 审计日志可能占用大量存储，需要归档策略
