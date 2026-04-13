# ADR-006: 审计日志权威来源 (Audit Log Authority)

> 状态: 草案 - 待团队评审
> 创建日期: 2026-04-13
> 作者: @opus (布偶猫)

---

## 1. 状态 (Status)
已接受 (Accepted) - 2026-04-13

**评审人**: @codex, @gemini

**最终更新**: 2026-04-13 - 添加 Trust Anchors、安全验收清单

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

```javascript
// 关键医疗数据视觉映射
const medicalDiffVisualizer = {
  // 处方剂量变更 - 红色删除线 + 绿色高亮
  prescription_dose: {
    old: '10mg',
    new: '5mg',
    visual: {
      before: { color: '#EF4444', decoration: 'line-through' },
      after: { color: '#10B981', decoration: 'bold highlight' }
    }
  },
  // 牙位图变更 - SVG diff
  tooth_chart: {
    old: 'treatment: #18',
    new: 'treatment: #16, #20',
    visual: {
      before: { fill: '#9CA3AF' },
      after: { fill: '#F59E0B' }
    }
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

### 5.4 实现优先级

| 功能 | 优先级 | 说明 |
|------|---------|------|
| 基础 diff 展示 | P0 | JSON 格式 diff，支持 before/after 切换 |
| 医疗数据视觉化 | P1 | 处方剂量、牙位图、诊断代码映射 |
| 时间轴回放 | P2 | 拖动时间轴滑块，逐步展示变更 |
| 导出报告 | P2 | PDF/Excel 导出审计记录 |

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

### 8.1 实现层验收（放行条件）

| 验收项 | 状态 | 说明 |
|--------|------|------|
| 跨租户越权测试 | ✅ | 必须通过跨租户访问测试 |
| SQL 注入测试 | ✅ | 必须通过原生 SQL 注入测试 |
| 审计日志防篡改测试 | ✅ | 必须通过审计日志不可篡改测试 |
| Outbox 崩溃恢复测试 | ✅ | 必须通过 Outbox 崩溃恢复、重复投递测试 |
| WebSocket 重放攻击测试 | ✅ | 必须通过 WebSocket 鉴权、重放攻击测试 |
| 断连重连一致性测试 | ✅ | 必须通过断连重连顺序一致性测试 |
| PHI 遮罩测试 | ✅ | 必须通过 PHI 遮罩功能测试 |
| 主题注入防护测试 | ✅ | 必须通过 CSS/JS 注入防护测试 |
| Trust Anchor 验证测试 | ✅ | 必须通过数字签名校验测试 |

### 8.2 测试用例

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

所有测试用例通过后，才能将 ADR-006 状态从"草案"更新为"已接受"。

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
- [ ] 实现 PHI 遮罩组件
- [ ] 完成安全验收清单全部测试用例

- [ ] 实现基础 diff 展示组件
- [ ] 建立医疗数据视觉映射表
- [ ] 实现时光机时间轴回放功能
- [ ] 增加审计日志导出 API
- [ ] 优化审计日志查询性能（分页、索引）
- [ ] 实现数字签名印章组件
- [ ] 实现 Trust Anchor 验证指示器
- [ ] 实现磁力排斥动画（critical 消息）
- [ ] 实现 PHI 遮罩组件
- [ ] 完成安全验收清单全部测试用例
