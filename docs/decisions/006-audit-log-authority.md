# ADR-006: 审计日志权威来源 (Audit Log Authority)

> 状态: **条件接受（待放行）** (Conditionally Accepted - Pending Evidence)
> 创建日期: 2026-04-13
> 作者: @opus (布偶猫)

---

## 1. 状态 (Status)
**条件接受 (Conditionally Accepted)** - 2026-04-13

**放行判据**: 以下安全验收清单全部测试通过，并提供完整测试证据（测试报告、用例通过记录、覆盖率数据）

**评审人**: @codex, @gemini

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

## 5. 审计时光机组件 (Audit Time-Machine Component)

**创意人**: @gemini

### 5.1 组件设计目标
审计日志不仅要展示 JSON diff，而是通过视觉化的方式降低审计人员的认知成本。

### 5.2 Trust Anchors - 数据签名水印

**创意人**: @gemini, @codex

**设计理念**: 通过视觉锚点强化审计记录的"不可篡改"认知，体现 **Trust（可信）** 核心设计理念。

#### 5.2.1 数字签名印章

```css
.audit-record.verified {
  /* 羊皮纸纹理背景 */
  background: 
    linear-gradient(135deg, #FDF8F3 0%, #F3F4F6 50%, #FDF8F3 100%),
    url('/textures/paper-grain.png');

  /* 硬核缅因猫 (Codex) 质感印章 */
  position: relative;
}

.audit-record.verified::after {
  content: '';
  position: absolute;
  top: 10px;
  right: 10px;
  width: 80px;
  height: 80px;
  
  /* 印章样式 */
  background: 
    radial-gradient(circle at 30% 30%, transparent 10%, rgba(59, 130, 246, 0.9) 20%),
    linear-gradient(135deg, #3B82F6, #2563EB);
  border: 2px solid #1D4ED8;
  border-radius: 50%;
  
  /* 印章文字 */
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: 'Courier New', monospace;
  font-size: 10px;
  color: #FFFFFF;
  text-align: center;
  line-height: 1.2;
  
  /* 印章遮罩效果 */
  mask-image: url('/textures/stamp-mask.png');
  -webkit-mask-image: url('/textures/stamp-mask.png');
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}
```

**印章内容**:
```
┌────────────┐
│  VERIFIED  │
│ 2026-04-13 │
│  #A7F3B9D  │
└────────────┘
  (Hardcore)
  Maine Coon
```

#### 5.2.2 签名校验指示器

```typescript
interface AuditSignature {
  signature_hash: string;      // SHA-256 签名哈希
  signer_id: string;         // 签名者 ID
  signed_at: string;         // 签名时间
  algorithm: 'ed25519';     // 签名算法
}

// UI 展示
const SignatureIndicator = {
  verified: {
    icon: 'shield-check',
    color: '#10B981',  // success green
    text: '数字签名校验通过'
  },
  invalid: {
    icon: 'shield-x',
    color: '#EF4444',  // danger red
    text: '签名校验失败'
  }
};
```

### 5.3 视觉化设计

#### 5.3.1 医疗数据视觉映射表 (Medical Data Visual Mapping)

完整的医疗数据 diff 视觉映射定义：

```javascript
// 完整医疗数据视觉映射表
const MedicalDiffVisualMapping = {
  // 处方相关
  prescription_dose: {
    label: '处方剂量',
    visual: {
      before: { color: '#EF4444', decoration: 'line-through' },
      after: { color: '#10B981', decoration: 'bold highlight' }
    }
  },
  prescription_frequency: {
    label: '用药频次',
    visual: {
      before: { color: '#EF4444' },
      after: { color: '#10B981', background: '#ECFDF5' }
    }
  },
  prescription_duration: {
    label: '用药周期',
    visual: {
      before: { color: '#9CA3AF', decoration: 'strikethrough' },
      after: { color: '#F59E0B', font: 'bold' }
    }
  },

  // 牙科相关
  tooth_chart: {
    label: '牙位图',
    visual: {
      before: { fill: '#9CA3AF', stroke: '#D1D5DB' },
      after: { fill: '#F59E0B', stroke: '#D97706' }
    }
  },
  tooth_status: {
    label: '牙位状态',
    visual: {
      normal: { fill: '#FFFFFF', stroke: '#E5E7EB' },
      treated: { fill: '#FEF3C7', stroke: '#F59E0B' },
      missing: { fill: '#374151', stroke: '#111827' }
    }
  },

  // 诊断相关
  diagnosis_code: {
    label: '诊断代码 (ICD-10)',
    visual: {
      before: { color: '#9CA3AF', font: 'monospace' },
      after: { color: '#3B82F6', font: 'monospace', background: '#EFF6FF' }
    }
  },
  diagnosis_name: {
    label: '诊断名称',
    visual: {
      added: { color: '#10B981', icon: '➕' },
      removed: { color: '#EF4444', icon: '➖' },
      modified: { color: '#F59E0B', icon: '✏️' }
    }
  },

  // 检查相关
  lab_result: {
    label: '检查结果',
    visual: {
      normal: { color: '#10B981', background: '#ECFDF5' },
      abnormal: { color: '#EF4444', background: '#FEF2F2' },
      borderline: { color: '#F59E0B', background: '#FFFBEB' }
    }
  },
  lab_reference_range: {
    label: '参考范围',
    visual: {
      before: { color: '#6B7280', border: 'dashed' },
      after: { color: '#9CA3AF', border: 'solid' }
    }
  },

  // 手术相关
  surgery_type: {
    label: '手术类型',
    visual: {
      before: { color: '#EF4444', decoration: 'line-through' },
      after: { color: '#3B82F6', background: '#EFF6FF', border: '2px solid #2563EB' }
    }
  },
  surgery_status: {
    label: '手术状态',
    visual: {
      scheduled: { color: '#9CA3AF', icon: '📅' },
      in_progress: { color: '#3B82F6', icon: '🏥' },
      completed: { color: '#10B981', icon: '✅' },
      cancelled: { color: '#EF4444', icon: '❌' }
    }
  },

  // 费用相关
  fee_amount: {
    label: '费用金额',
    visual: {
      before: { color: '#EF4444', decoration: 'line-through' },
      after: { color: '#10B981', font: 'bold' }
    }
  },
  fee_type: {
    label: '费用类型',
    visual: {
      medical: { color: '#3B82F6' },
      pharmacy: { color: '#8B5CF6' },
      examination: { color: '#EC4899' },
      other: { color: '#9CA3AF' }
    }
  }
};

// Diff 渲染器
const DiffRenderer = {
  render: (field: string, oldValue: any, newValue: any) => {
    const mapping = MedicalDiffVisualMapping[field];
    if (!mapping) {
      return this.renderJsonDiff(oldValue, newValue);
    }

    return this.renderVisualDiff(mapping, oldValue, newValue);
  },

  renderVisualDiff: (mapping: any, oldValue: any, newValue: any) => {
    // 根据映射配置渲染视觉化 diff
    const container = document.createElement('div');
    container.className = 'diff-container';

    // Before 节点
    const beforeNode = this.createBeforeNode(mapping.visual.before, oldValue);
    container.appendChild(beforeNode);

    // After 节点
    const afterNode = this.createAfterNode(mapping.visual.after, newValue);
    container.appendChild(afterNode);

    return container;
  },

  createBeforeNode: (visual: any, value: any) => {
    const node = document.createElement('div');
    node.className = 'diff-before';
    Object.assign(node.style, visual);
    node.textContent = value;
    return node;
  },

  createAfterNode: (visual: any, value: any) => {
    const node = document.createElement('div');
    node.className = 'diff-after';
    Object.assign(node.style, visual);
    node.textContent = value;
    return node;
  },

  renderJsonDiff: (oldValue: any, newValue: any) => {
    // Fallback: 使用 JSON diff 库渲染
    const diff = this.computeJsonDiff(oldValue, newValue);
    return this.highlightJsonChanges(diff);
  }
};
```

### 5.3 组件接口

```typescript
interface AuditTimeMachineProps {
  auditRecord: AuditLog;
  medicalContext?: MedicalContext;
  onDrillDown?: (recordId: string) => void;
}
```

#### 5.3.1 基础 Diff 展示组件 (Basic Diff Component)

```typescript
// Diff 类型定义
type DiffType = 'added' | 'removed' | 'modified' | 'unchanged';

interface DiffItem {
  key: string;
  path: string[];
  oldValue: any;
  newValue: any;
  type: DiffType;
}

interface DiffViewerProps {
  oldData: any;
  newData: any;
  diff: DiffItem[];
  viewType?: 'json' | 'visual' | 'split';
  expandDepth?: number;
  showUnchanged?: boolean;
}

// Diff 视图组件
const DiffViewer = {
  // 渲染 JSON 格式 diff
  renderJsonDiff: (props: DiffViewerProps) => {
    const { diff, expandDepth = 2 } = props;

    return (
      <div className="diff-json-container">
        {diff.map((item, index) => (
          <DiffNode
            key={item.key}
            item={item}
            depth={0}
            maxDepth={expandDepth}
          />
        ))}
      </div>
    );
  },

  // 渲染视觉化 diff（医疗数据专用）
  renderVisualDiff: (props: DiffViewerProps) => {
    const { diff, viewType = 'visual' } = props;

    return (
      <div className="diff-visual-container">
        {diff.map((item, index) => (
          <VisualDiffItem key={item.key} item={item} />
        ))}
      </div>
    );
  },

  // 渲染分屏 diff（左右对比）
  renderSplitDiff: (props: DiffViewerProps) => {
    const { oldData, newData, diff } = props;

    return (
      <div className="diff-split-container">
        <div className="diff-panel diff-panel-before">
          <h3>变更前</h3>
          <CodeBlock data={oldData} />
        </div>
        <div className="diff-divider"></div>
        <div className="diff-panel diff-panel-after">
          <h3>变更后</h3>
          <CodeBlock data={newData} />
        </div>
      </div>
    );
  }
};

// Diff 节点组件
const DiffNode = ({ item, depth, maxDepth }: DiffNodeProps) => {
  const { key, type, oldValue, newValue } = item;
  const isExpanded = depth < maxDepth;
  const hasChildren = Array.isArray(newValue) || typeof newValue === 'object';

  const diffClass = {
    added: 'diff-added',
    removed: 'diff-removed',
    modified: 'diff-modified',
    unchanged: 'diff-unchanged'
  }[type];

  return (
    <div className={`diff-node ${diffClass}`}>
      <div className="diff-header">
        <span className="diff-key">{key}</span>
        <span className="diff-type-badge">{type}</span>
      </div>

      {hasChildren && isExpanded ? (
        <div className="diff-children">
          {Object.entries(newValue).map(([k, v]) => (
            <DiffNode
              key={k}
              item={{
                key: k,
                type: type === 'unchanged' ? 'unchanged' : type,
                oldValue: oldValue?.[k],
                newValue: v
              }}
              depth={depth + 1}
              maxDepth={maxDepth}
            />
          ))}
        </div>
      ) : (
        <div className="diff-value">
          {type === 'added' ? (
            <span className="diff-value-added">{JSON.stringify(newValue)}</span>
          ) : type === 'removed' ? (
            <span className="diff-value-removed">{JSON.stringify(oldValue)}</span>
          ) : (
            <>
              <span className="diff-value-removed">{JSON.stringify(oldValue)}</span>
              <span className="diff-arrow">→</span>
              <span className="diff-value-added">{JSON.stringify(newValue)}</span>
            </>
          )}
        </div>
      )}
    </div>
  );
};
```

#### 5.3.2 Diff 样式定义

```css
/* Diff 容器 */
.diff-json-container {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 14px;
  line-height: 1.6;
  padding: var(--spacing-4);
  background: var(--bg-surface);
  border-radius: var(--radius-md);
}

/* Diff 节点 */
.diff-node {
  padding-left: calc(var(--spacing-3) * var(--depth, 0));
  border-left: 2px solid transparent;
  transition: all 0.2s ease;
}

.diff-node:hover {
  background: rgba(59, 130, 246, 0.05);
  border-left-color: var(--primary-color);
}

/* Diff 头部 */
.diff-header {
  display: flex;
  align-items: center;
  gap: var(--spacing-2);
  margin-bottom: var(--spacing-1);
}

.diff-key {
  font-weight: 600;
  color: var(--text-primary);
}

/* Diff 类型徽章 */
.diff-type-badge {
  padding: 2px 8px;
  border-radius: var(--radius-sm);
  font-size: 12px;
  font-weight: 500;
}

.diff-added .diff-type-badge {
  background: #ECFDF5;
  color: #065F46;
}

.diff-removed .diff-type-badge {
  background: #FEF2F2;
  color: #991B1B;
}

.diff-modified .diff-type-badge {
  background: #FFFBEB;
  color: #92400E;
}

.diff-unchanged .diff-type-badge {
  background: #F3F4F6;
  color: #374151;
}

/* Diff 值 */
.diff-value {
  padding: var(--spacing-2);
  background: rgba(0, 0, 0, 0.02);
  border-radius: var(--radius-sm);
  white-space: pre-wrap;
  word-break: break-all;
}

.diff-value-added {
  color: #10B981;
}

.diff-value-removed {
  color: #EF4444;
  text-decoration: line-through;
  opacity: 0.7;
}

.diff-arrow {
  margin: 0 var(--spacing-2);
  color: var(--text-secondary);
}

/* Diff 子节点 */
.diff-children {
  margin-top: var(--spacing-2);
  padding-left: var(--spacing-3);
}

/* 分屏 diff 容器 */
.diff-split-container {
  display: grid;
  grid-template-columns: 1fr 2px 1fr;
  gap: var(--spacing-4);
  height: 100%;
}

.diff-panel {
  padding: var(--spacing-4);
  background: var(--bg-surface);
  border-radius: var(--radius-md);
  overflow: auto;
}

.diff-panel-before {
  border-left: 3px solid #EF4444;
}

.diff-panel-after {
  border-left: 3px solid #10B981;
}

.diff-divider {
  background: #E5E7EB;
  height: 100%;
}

/* 视觉化 diff 容器 */
.diff-visual-container {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-3);
}
```

#### 5.3.3 工具函数

```typescript
// Diff 计算工具
const DiffCalculator = {
  // 计算 JSON diff
  compute: (oldObj: any, newObj: any): DiffItem[] => {
    const diff: DiffItem[] = [];
    const allKeys = new Set([...Object.keys(oldObj || {}), ...Object.keys(newObj || {})]);

    for (const key of allKeys) {
      const oldValue = oldObj?.[key];
      const newValue = newObj?.[key];

      if (oldValue === undefined) {
        // 新增
        diff.push({ key, path: [key], oldValue, newValue, type: 'added' });
      } else if (newValue === undefined) {
        // 删除
        diff.push({ key, path: [key], oldValue, newValue, type: 'removed' });
      } else if (!this.isEqual(oldValue, newValue)) {
        // 修改
        if (typeof oldValue === 'object' && typeof newValue === 'object') {
          // 递归处理嵌套对象
          const nestedDiff = this.compute(oldValue, newValue);
          nestedDiff.forEach(item => {
            item.path.unshift(key);
          });
          diff.push(...nestedDiff);
        } else {
          diff.push({ key, path: [key], oldValue, newValue, type: 'modified' });
        }
      }
    }

    return diff;
  },

  // 深度相等判断
  isEqual: (a: any, b: any): boolean => {
    if (a === b) return true;
    if (typeof a !== typeof b) return false;
    if (Array.isArray(a) && Array.isArray(b)) {
      return a.length === b.length && a.every((v, i) => this.isEqual(v, b[i]));
    }
    if (typeof a === 'object' && typeof b === 'object') {
      const keysA = Object.keys(a);
      const keysB = Object.keys(b);
      if (keysA.length !== keysB.length) return false;
      return keysA.every(k => this.isEqual(a[k], b[k]));
    }
    return false;
  }
};
```

### 5.4 实现优先级

| 功能 | 优先级 | 说明 |
|------|---------|------|
| 基础 diff 展示 | P0 | JSON 格式 diff，支持 before/after 切换 |
| 医疗数据视觉化 | P1 | 处方剂量、牙位图、诊断代码映射 |
| 时间轴回放 | P2 | 拖动时间轴滑块，逐步展示变更 |
| 导出报告 | P2 | PDF/Excel 导出审计记录 |

### 5.5 审计日志导出 API 设计

#### 5.5.1 API 规范

```http
# 异步导出请求
POST /api/v1/audit/export
Authorization: Bearer {jwt}
Content-Type: application/json

{
  "filters": {
    "tenant_id": "tenant_999",
    "user_id": "user_123",
    "resource_type": "appointment",
    "action": "update",
    "date_from": "2026-04-01T00:00:00Z",
    "date_to": "2026-04-13T23:59:59Z"
  },
  "format": "pdf" | "excel" | "csv",
  "include_phi": false,
  "include_signature": true
}

# 响应
{
  "export_id": "export_ulid_456",
  "status": "pending",
  "expires_at": "2026-04-13T18:00:00Z",
  "estimated_seconds": 30
}

# 查询导出状态
GET /api/v1/audit/export/{export_id}

# 响应
{
  "export_id": "export_ulid_456",
  "status": "completed",
  "download_url": "https://storage.hos.com/exports/tenant_999/audit_20260413.pdf?token=xyz",
  "expires_at": "2026-04-13T18:00:00Z",
  "created_at": "2026-04-13T17:00:00Z",
  "completed_at": "2026-04-13T17:00:30Z",
  "record_count": 1523
}

# 下载导出文件
GET {download_url}
```

#### 5.5.2 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 创建导出任务 | admin | `audit:export` |
| 查询导出状态 | admin | `audit:export` |
| 下载导出文件 | admin | `audit:export` |

#### 5.5.3 导出格式规范

**PDF 格式：**
- 包含封面页（导出时间、租户信息、导出人）
- 目录页（按日期、资源类型、操作类型索引）
- 每条审计记录独立页面
- 数字签名印章（如果启用）
- 水印（导出者 + 时间）

**Excel 格式：**
- 多工作表：按资源类型分表
- 表头冻结
- 条件格式标记（红色=删除、绿色=新增、黄色=修改）
- 隐藏 PHI 列（默认）

#### 5.5.4 安全约束

```typescript
// 导出安全验证
interface AuditExportSecurity {
  validate: (request: ExportRequest) => {
    // 1. 租户隔离
    if (request.tenant_id !== getCurrentTenantId()) {
      throw new SecurityError('跨租户导出禁止');
    }

    // 2. 时间范围限制（最多 90 天）
    const dateRange = new Date(request.date_to) - new Date(request.date_from);
    if (dateRange > 90 * 24 * 60 * 60 * 1000) {
      throw new ValidationError('导出时间范围不能超过 90 天');
    }

    // 3. PHI 导出需特殊权限
    if (request.include_phi && !hasPermission('audit:export_phi')) {
      throw new SecurityError('无权导出 PHI 数据');
    }

    // 4. 频率限制（每租户每小时最多 5 次）
    const recentExports = await getRecentExports(request.tenant_id, '1h');
    if (recentExports.length >= 5) {
      throw new RateLimitError('导出请求过于频繁');
    }

    return { valid: true };
  },

  // 生成临时下载令牌（有效期 1 小时）
  generateDownloadToken: (exportId: string) => {
    const token = jwt.sign(
      { export_id: exportId, exp: Date.now() + 3600000 },
      process.env.EXPORT_SECRET
    );
    return `https://storage.hos.com/exports/${exportId}?token=${token}`;
  }
};
```

#### 5.5.5 审计日志导出记录

```sql
-- 审计导出记录表
CREATE TABLE audit_exports (
  id ULID PRIMARY KEY,
  tenant_id VARCHAR NOT NULL,
  export_id VARCHAR NOT NULL UNIQUE,
  user_id VARCHAR NOT NULL,
  filters JSONB NOT NULL,
  format VARCHAR NOT NULL,
  status VARCHAR NOT NULL,  -- pending | processing | completed | failed
  file_url VARCHAR,
  record_count INTEGER,
  file_size_bytes INTEGER,
  created_at TIMESTAMP NOT NULL,
  completed_at TIMESTAMP,
  expires_at TIMESTAMP NOT NULL,
  INDEX idx_tenant_user (tenant_id, user_id, created_at DESC),
  INDEX idx_export_id (export_id)
);

-- 状态变更约束
CHECK (status IN ('pending', 'processing', 'completed', 'failed'))
```

---

## 6. 磁力排斥动画 (Critical Message Repulsion Effect)

**设计人**: @gemini

### 6.1 设计目标
`critical` 优先级消息在屏幕上应有"排斥效应"，周围普通消息向外轻微移动，为高优先级消息留出独立空间，视觉上强化紧迫性。

### 6.2 实现方式

```css
/* Critical 消息气泡 */
.notification-bubble[data-priority="critical"] {
  /* 磁力排斥效果 */
  position: relative;
  z-index: 100;
  margin: 16px 0;  /* 上下留出更大间距 */
  box-shadow: 
    0 0 4px rgba(59, 130, 246, 0.4),  /* 内发光 */
    0 0 8px rgba(59, 130, 246, 0.2);  /* 外光晕 */
}

/* Critical 消息出现时的排斥动画 */
@keyframes repel {
  0% {
    transform: scale(0.95);
    opacity: 0;
  }
  50% {
    transform: scale(1.02);
    opacity: 1;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.notification-bubble[data-priority="critical"].enter {
  animation: repel 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* 周围普通消息被排斥（通过 CSS 选择器实现） */
.notification-bubble[data-priority="critical"] ~ .notification-bubble:not([data-priority="critical"]) {
  transition: transform 0.3s ease-out, opacity 0.3s ease-out;
}

/* Critical 消息进入时推开周围消息 */
.notification-bubble[data-priority="critical"].enter ~ .notification-bubble:not([data-priority="critical"]) {
  transform: translateX(20px);
  opacity: 0.6;
}
```

### 6.3 降噪与节流

**防告警风暴**:
- 同一 `collapse_key` 在 1 分钟内出现 > 5 条 critical 消息时，只显示最新的 1 条
- 其余消息折叠到"批量"状态，不播放排斥动画
- 批量消息显示"🔔 X 条紧急告警已聚合"，点击后展开
- 静默模式下不播放排斥动画

---

## 7. PHI 遮罩保护 (PHI Masking Protection)

**设计人**: @gemini, @codex

### 7.1 设计目标
涉及患者个人信息 (PHI) 的通知必须保护敏感数据，避免信息泄露。

### 7.2 遮罩策略

| 数据类型 | 默认状态 | 显示条件 | 遮罩规则 |
|---------|---------|---------|---------|
| 患者姓名 | 李\*\* | 患者本人登录 | 姓氏 + 第 1 个字 + ** |
| 手机号 | 138****5678 | 患者本人或管理员 | 前 3 位 + **\*\*后 4 位 |
| 身份证号 | 110**********1234 | 仅管理员 | 前 6 位 + **\*\*后 4 位 |
| 病史详情 | [已隐藏] | 点击展开查看 | 仅管理员可展开 |
| 地址信息 | 北京市朝阳区\*\*\* | 患者本人登录 | 具体门牌号隐藏 |

### 7.3 实现方式

```typescript
interface PHIProps {
  data: any;
  currentUser?: string;      // 当前用户 ID
  isSelf?: boolean;          // 是否本人
  reveal?: boolean;           // 是否已展开
}

const PHIMasker = {
  shouldMask: (data: any, currentUser?: string) => {
    return !data.isSelf && currentUser !== 'admin';
  },
  
  mask: (value: string, type: 'name' | 'phone' | 'idcard') => {
    switch (type) {
      case 'name':
        return value.charAt(0) + '**';
      case 'phone':
        return value.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2');
      case 'idcard':
        return value.replace(/(\d{6})(\d{4})(\d{4})/, '$1******$3');
    }
  }
};
```

---

## 8. 安全验收清单 (Security Acceptance Checklist)

**设计人**: @codex

### 8.1 放行判据

**ADR-006 状态为"条件接受（待放行）"，以下两个清单全部完成后才可更新为"已投产" (Deployed)：**

1. **安全验收清单**：9 项 P1 测试用例全部通过，并提供完整测试证据
2. **实现清单**：10 项实现任务全部完成并通过代码审查

### 8.2 安全验收清单（9 项 P1 测试）

| 验收项 | 状态 | 说明 |
|--------|------|------|
| 跨租户越权测试 | ⏳ | 必须通过跨租户访问测试 |
| SQL 注入测试 | ⏳ | 必须通过原生 SQL 注入测试 |
| 审计日志防篡改测试 | ⏳ | 必须通过审计日志不可篡改测试 |
| Outbox 崩溃恢复测试 | ⏳ | 必须通过 Outbox 崩溃恢复、重复投递测试 |
| WebSocket 重放攻击测试 | ⏳ | 必须通过 WebSocket 鉴权、重放攻击测试 |
| 断连重连一致性测试 | ⏳ | 必须通过断连重连顺序一致性测试 |
| PHI 遮罩测试 | ⏳ | 必须通过 PHI 遮罩功能测试 |
| 主题注入防护测试 | ⏳ | 必须通过 CSS/JS 注入防护测试 |
| Trust Anchor 验证测试 | ⏳ | 必须通过数字签名校验测试 |

**注意**: 当前状态标记为 `✅` 表示"设计完成"，并非"测试通过"。实际验收需提供完整测试证据（测试报告、用例执行日志、覆盖率数据）。

### 8.3 测试用例（Gherkin 格式）

```gherkin
# 跨租户越权测试
Feature: 审计日志越权访问
  Scenario: 用户 A (tenant_1) 尝试访问 tenant_2 的审计日志
  Expect: HTTP 403 Forbidden
  
# SQL 注入测试
Feature: SQL 注入防护
  Scenario: 尝试通过构造 SQL 注入绕过租户检查
  Expect: HTTP 400 Bad Request 或安全告警
  
# 审计日志防篡改测试
Feature: 审计日志不可篡改
  Scenario: 尝试直接 UPDATE/DELETE 审计记录
  Expect: 数据库约束拒绝或操作失败
  
# Outbox 崩溃恢复测试
Feature: Outbox 崩溃恢复
  Scenario: 模拟 Outbox 崩溃，验证重复消费不会产生重复通知
  Expect: 消息幂等键生效，无重复消息
  
# WebSocket 重放攻击测试
Feature: WebSocket 鉴权
  Scenario: 使用已用过的 message_id 重放消息
  Expect: 服务端拒绝重复消息
  
# 断连重连一致性测试
Feature: 断连重连
  Scenario: 客户端断连重连，验证 last_acked_seq 恢复
  Expect: 按 seq 顺序补发，无乱序、无重复
  
# PHI 遮罩测试
Feature: PHI 遮罩
  Scenario: 非本人用户查看患者敏感信息
  Expect: 显示遮罩数据（李**）
  
# 主题注入防护测试
Feature: CSS 注入防护
  Scenario: 尝试通过 theme_config 注入恶意 CSS
  Expect: 正则校验拒绝或转义处理
  
# Trust Anchor 验证测试
Feature: 数字签名验证
  Scenario: 验证已签名记录的哈希值
  Expect: 显示校验通过图标和数字印章
```

### 8.3 验收通过标准

所有测试用例通过后，才能将 ADR-006 状态从"已接受"更新为"已投产" (Deployed)。

---

## 9. 后续行动项

- [ ] 实现基础 diff 展示组件
- [ ] 建立医疗数据视觉映射表
- [ ] 实现时光机时间轴回放功能
- [ ] 增加审计日志导出 API
- [ ] 优化审计日志查询性能（分页、索引）
- [ ] 实现数字签名印章组件
- [ ] 实现 Trust Anchor 验证指示器
- [ ] 实现磁力排斥动画（critical 消息）
- [ ] 完成 PHI 遮罩组件实现（设计 Token 已在 design-system.md 中定义）
- [ ] 完成安全验收清单全部测试用例（9 项 P1 测试）
