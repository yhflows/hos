# 设计系统规范 (Design System Specification)

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫), @gemini (暹罗猫)
> 创建日期: 2026-04-13

---

## 1. 设计理念

HOS 医疗系统以 **"温暖、阳光、专业"** 为核心视觉基调，传递"关怀、可靠、科技"的品牌感知。

**关键词**: Calm（冷静）、Warm（温暖）、Professional（专业）、Trust（可信）

---

## 2. 色彩系统 (Color System)

### 2.1 主色调 (Primary Colors)

```css
:root {
  /* 猫咖暖色系 - 温暖、阳光 */
  --bg-app: #FDF8F3;
  --bg-surface: #FFFFFF;
  --text-primary: #1F2937;
  --text-secondary: #6B7280;
  --text-tertiary: #9CA3AF;

  /* 品牌色 - 医疗蓝 */
  --primary-color: #3B82F6;
  --primary-hover: #2563EB;
  --primary-active: #1D4ED8;

  /* 功能色 */
  --success-color: #10B981;
  --warning-color: #F59E0B;
  --danger-color: #EF4444;
  --info-color: #3B82F6;
}
```

### 2.2 语义色 (Semantic Colors)

| 用途 | 色值 | 说明 |
|------|------|------|
| 医疗蓝 | `#3B82F6` | 主要操作、链接、高亮 |
| 成功绿 | `#10B981` | 成功状态、完成标记 |
| 警告橙 | `#F59E0B` | 警告提示、待处理 |
| 危险红 | `#EF4444` | 错误、删除、危险操作 |
| 信息蓝 | `#60A5FA` | 信息提示、辅助说明 |
| 中性灰 | `#9CA3AF` | 禁用状态、次要文本 |
| 背景暖 | `#FDF8F3` | 应用背景、卡片底色 |
| 背景纯白 | `#FFFFFF` | 表面、弹窗背景 |

---

## 3. 间距系统 (Spacing System)

基于 **8px 基准单位**，确保视觉一致性。

```css
:root {
  --spacing-0: 0;
  --spacing-1: 4px;
  --spacing-2: 8px;
  --spacing-3: 12px;
  --spacing-4: 16px;
  --spacing-5: 20px;
  --spacing-6: 24px;
  --spacing-8: 32px;
  --spacing-10: 40px;
  --spacing-12: 48px;
}
```

### 3.1 使用场景

| Token | 用途 |
|-------|------|
| `--spacing-1` | 元素内边距、小间隙 |
| `--spacing-2` | 卡片内边距、表单控件间距 |
| `--spacing-3` | 区块间距 |
| `--spacing-4` | 容器外边距、页面边距 |
| `--spacing-6` | 大区块间距、模态框间距 |

---

## 4. 圆角系统 (Border Radius)

```css
:root {
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-full: 9999px;
}
```

### 4.1 使用场景

| Token | 用途 |
|-------|------|
| `--radius-sm` | 按钮、输入框、标签 |
| `--radius-md` | 卡片、弹窗（默认） |
| `--radius-lg` | 大卡片、面板 |
| `--radius-xl` | 模态框、抽屉 |
| `--radius-full` | 圆形头像、标签 |

---

## 5. 阴影系统 (Box Shadow)

```css
:root {
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.07);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 25px rgba(0, 0, 0, 0.1);
}
```

---

## 6. 字体系统 (Typography)

### 6.1 字体栈

```css
:root {
  --font-family-base: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-family-mono: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
}
```

### 6.2 字号层级

```css
:root {
  --text-xs: 12px;
  --text-sm: 14px;
  --text-base: 16px;
  --text-lg: 18px;
  --text-xl: 20px;
  --text-2xl: 24px;
  --text-3xl: 30px;
}
```

---

## 7. 组件规范 (Component Guidelines)

### 7.1 通知气泡 (Notification Bubble)

#### 优先级视觉

| 优先级 | 背景色 | 边框 | 动效 |
|-------|--------|------|------|
| `critical` | `#FEF2F2` + 红色边框 | 硬核 2px | 脉冲 + 强制置顶 |
| `normal` | `#EFF6FF` + 蓝色边框 | 暹罗猫 1px | 潜微渐变 |
| `ephemeral` | `#F3F4F6` + 灰色边框 | 淡入 1px | 淡出动画 |

### 7.2 卡片组件 (Card)

```css
.card {
  background: var(--bg-surface);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-md);
  padding: var(--spacing-4);
  border: 1px solid rgba(0, 0, 0, 0.05);
}
```

### 7.3 按钮组件 (Button)

```css
.button-primary {
  background: var(--primary-color);
  color: #FFFFFF;
  border-radius: var(--radius-sm);
  padding: var(--spacing-2) var(--spacing-4);
  font-size: var(--text-sm);
  transition: all 0.2s ease;
}

.button-primary:hover {
  background: var(--primary-hover);
  transform: translateY(-1px);
}
```

### 7.4 PHI 遮罩组件 (PHI Masking Component)

遵循 ADR-006 审计日志权威来源，对患者个人信息 (PHI) 进行可视化保护。

#### 7.4.1 遮罩视觉 Token

```css
:root {
  /* PHI 遮罩颜色 */
  --phi-mask-bg: #F3F4F6;
  --phi-mask-text: #9CA3AF;
  --phi-mask-border: #D1D5DB;
  --phi-reveal-bg: #FEF3C7;
  --phi-reveal-text: #92400E;
  
  /* 遮罩动画 */
  --phi-mask-transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}
```

#### 7.4.2 遮罩组件样式

```css
/* 基础遮罩容器 */
.phi-mask {
  background: var(--phi-mask-bg);
  color: var(--phi-mask-text);
  border: 1px solid var(--phi-mask-border);
  border-radius: var(--radius-sm);
  padding: 2px 6px;
  font-family: 'Courier New', monospace;
  transition: var(--phi-mask-transition);
  cursor: pointer;
  position: relative;
  display: inline-block;
}

/* 已展开状态 */
.phi-mask.revealed {
  background: var(--phi-reveal-bg);
  color: var(--phi-reveal-text);
  border-color: #F59E0B;
}

/* 悬停效果 */
.phi-mask:hover {
  background: #E5E7EB;
  border-color: #9CA3AF;
}

/* 点击展开后的锁定指示器 */
.phi-mask.revealed::after {
  content: '🔓';
  position: absolute;
  top: -8px;
  right: -8px;
  font-size: 10px;
  animation: bounce-in 0.3s ease;
}

@keyframes bounce-in {
  0% { transform: scale(0); opacity: 0; }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); opacity: 1; }
}
```

#### 7.4.3 遮罩策略映射

| 数据类型 | 默认状态 | 展开条件 | Token |
|---------|---------|---------|-------|
| 患者姓名 | `李**` | 本人/管理员 | `--phi-mask-name` |
| 手机号 | `138****5678` | 本人/管理员 | `--phi-mask-phone` |
| 身份证号 | `110**********1234` | 仅管理员 | `--phi-mask-idcard` |
| 病史详情 | `[已隐藏]` | 管理员展开 | `--phi-mask-history` |
| 地址信息 | `北京市朝阳区***` | 本人/管理员 | `--phi-mask-address` |

#### 7.4.4 组件接口

```typescript
interface PHIMaskProps {
  value: string;           // 原始值
  type: 'name' | 'phone' | 'idcard' | 'history' | 'address';
  currentUser?: string;     // 当前用户 ID
  isSelf?: boolean;        // 是否本人
  isAdmin?: boolean;       // 是否管理员
  onReveal?: () => void;  // 展开回调
  onLog?: (action: string) => void;  // 审计日志记录
}

// 遮罩工具函数
const PHIMasker = {
  mask: (value: string, type: keyof typeof PHIMasker.patterns): string => {
    const pattern = PHIMasker.patterns[type];
    return value.replace(pattern.regex, pattern.replacement);
  },

  patterns: {
    name: { regex: /(.{1}).+/, replacement: '$1**' },
    phone: { regex: /(\d{3})\d{4}(\d{4})/, replacement: '$1****$2' },
    idcard: { regex: /(\d{6}).+(\d{4})/, replacement: '$1******$2' },
    address: { regex: /(.*[市区县]).+/, replacement: '$1***' },
    history: { regex: /.+/, replacement: '[已隐藏]' }
  },

  // 解罩授权检查（完整闭环）
  canReveal: (props: PHIMaskProps): { allowed: boolean, reason?: string } => {
    // 1. 权限校验
    if (!props.isSelf && !props.isAdmin) {
      return { allowed: false, reason: '无权限：非本人且非管理员' };
    }

    // 2. 敏感数据类型特殊处理
    if (props.type === 'idcard' && !props.isAdmin) {
      return { allowed: false, reason: '身份证号仅管理员可查看' };
    }

    // 3. 二次确认要求（非本人展开需要审批）
    if (!props.isSelf && props.isAdmin) {
      return {
        allowed: true,
        reason: '需提供理由码并获得二次确认后允许'
      };
    }

    return { allowed: true };
  },

  // 理由码定义（用于审计追溯）
  reasonCodes: {
    CLINICAL_NEED: 'clinical',      // 临床诊疗需要
    INSURANCE_CLAIM: 'insurance',     // 保险理赔需要
    LEGAL_COMPLIANCE: 'legal',       // 法律合规需要
    PATIENT_REQUEST: 'patient_request', // 患者本人请求
    SYSTEM_ADMIN: 'system_admin'       // 系统管理需要
  },

  // 展开权限等级
  revealLevels: {
    SELF: 'self',           // 本人：直接展开
    ADMIN: 'admin',         // 管理员：需理由码
    SUPER_ADMIN: 'super'      // 超级管理员：无需理由（需记录）
  }
};
```

#### 7.4.5 审计日志集成

```typescript
// PHI 解罩完整流程
const handlePHIReveal = async (props: PHIMaskProps) => {
  // 1. 权限检查
  const authResult = PHIMasker.canReveal(props);
  if (!authResult.allowed) {
    // 记录拒绝访问审计
    await auditLogger.log({
      action: 'phi_reveal_denied',
      resource_type: props.type,
      user_id: props.currentUser,
      metadata: {
        denial_reason: authResult.reason,
        attempted_value_length: props.value?.length
      }
    });
    throw new SecurityError(authResult.reason);
  }

  // 2. 非本人展开需要理由码和二次确认
  if (!props.isSelf) {
    const reasonCode = await promptReasonCode();
    if (!reasonCode) {
      throw new ValidationError('请选择展开理由');
    }

    // 二次确认弹窗
    const confirmed = await showConfirmDialog({
      title: '确认展开患者敏感信息',
      message: `您即将查看 ${props.type} 类型的 PHI 数据`,
      warning: '此操作将被记录到审计日志',
      requirePassword: true
    });

    if (!confirmed) {
      // 记录取消展开审计
      await auditLogger.log({
        action: 'phi_reveal_cancelled',
        resource_type: props.type,
        user_id: props.currentUser,
        metadata: {
          reason_code: reasonCode,
          cancelled_at: new Date().toISOString()
        }
      });
      return;
    }
  }

  // 3. 记录审计日志（完整字段）
  await auditLogger.log({
    action: 'phi_reveal_success',
    resource_type: props.type,
    user_id: props.currentUser,
    metadata: {
      revealed_value_length: props.value.length,
      reason_code: props.isSelf ? 'patient_request' : props.reasonCode,
      reveal_level: props.isSelf ? PHIMasker.revealLevels.SELF : PHIMasker.revealLevels.ADMIN,
      patient_id: props.patientId,
      request_ip: getClientIP(),
      request_ua: getUserAgent(),
      confirmation_method: props.isSelf ? 'direct' : 'password',
      session_id: getSessionId()
    }
  });

  props.onReveal?.();
};

// 理由码选择组件
const ReasonCodeSelector = {
  codes: Object.values(PHIMasker.reasonCodes),

  labels: {
    clinical: '临床诊疗需要',
    insurance: '保险理赔需要',
    legal: '法律合规需要',
    patient_request: '患者本人请求',
    system_admin: '系统管理需要'
  },

  render: () => (
    <div className="reason-selector">
      <h4>请选择展开理由</h4>
      {Object.entries(ReasonCodeSelector.labels).map(([code, label]) => (
        <button key={code} value={code}>
          {label}
        </button>
      ))}
    </div>
  )
};

// 二次确认弹窗
const ConfirmDialog = {
  show: (props: ConfirmDialogProps) => {
    const { title, message, warning, requirePassword } = props;

    return (
      <Modal title={title}>
        <p>{message}</p>
        {warning && <Alert type="warning">{warning}</Alert>}
        {requirePassword && (
          <Input type="password" placeholder="请输入密码确认" />
        )}
        <div className="dialog-actions">
          <Button type="default">取消</Button>
          <Button type="primary" danger>确认展开</Button>
        </div>
      </Modal>
    );
  }
};
```

---

## 8. 租户主题定制 (Tenant Theme Customization)

遵循 ADR-003，仅支持 CSS 变量白名单注入。

### 8.1 允许的定制变量

```css
:root {
  /* 品牌色 */
  --primary-color: #3B82F6;           /* 可定制 */
  --logo-url: url('...');             /* 可定制 */

  /* 尺寸调整 */
  --border-radius: 8px;               /* 可定制 */
  --font-scale: 1.0;                 /* 可定制 */

  /* 对比度约束 */
  --contrast-min: 4.5;               /* 最小对比度 */
  --contrast-max: 7.0;               /* 最大对比度 */
}
```

### 8.2 实时主题预览 (@gemini 建议)

```typescript
// 租户后台主题配置组件
interface ThemePreviewProps {
  themeConfig: ThemeConfig;
  onPreview: (config: ThemeConfig) => void;
  onSave: (config: ThemeConfig) => void;
}

const components: {
  ThemePreviewPanel: {
    description: "实时预览主题配置，所见即所得",
    features: [
      "配色实时切换",
      "Logo 上传与预览",
      "圆角调整滑块",
      "对比度自动检测"
    ]
  }
};
```

---

## 9. Tailwind 配置映射

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        'cat-cafe-bg': 'var(--bg-app)',
        'cat-cafe-surface': 'var(--bg-surface)',
        'primary': 'var(--primary-color)',
        'primary-hover': 'var(--primary-hover)',
        'success': 'var(--success-color)',
        'warning': 'var(--warning-color)',
        'danger': 'var(--danger-color)',
      },
      spacing: {
        '1': 'var(--spacing-1)',
        '2': 'var(--spacing-2)',
        '3': 'var(--spacing-3)',
        '4': 'var(--spacing-4)',
      },
      borderRadius: {
        'sm': 'var(--radius-sm)',
        'md': 'var(--radius-md)',
        'lg': 'var(--radius-lg)',
        'xl': 'var(--radius-xl)',
      }
    }
  }
}
```

---

## 10. 后续行动项

- [ ] 建立 Storybook 组件库
- [ ] 实现主题预览组件
- [ ] 建立设计 token 与 Tailwind 的自动化映射
- [ ] 制定可访问性 (Accessibility) 规范

---

## 11. 高级视觉特效 (Advanced Visual Effects)

**设计人**: @gemini

### 11.1 呼吸感微动效 (Breathing Ripple Effect)

卡片边缘的淡蓝色波纹扩散效果，营造"生命力"和"温暖"的视觉体验。

```css
/* 呼吸感容器 */
.breathing-ripple-container {
  position: relative;
  overflow: hidden;
}

/* 波纹动画 */
@keyframes ripple-spread {
  0% {
    opacity: 0;
    transform: scale(0.8);
  }
  100% {
    opacity: 1;
    transform: scale(1.2);
  }
}

.breathing-ripple {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(circle, rgba(244, 230, 160, 0.4) 0%, rgba(244, 230, 160, 0.6));
  border-radius: 50%;
  transform: translate(-50%, -50%);
  animation: ripple-spread 2s ease-out;
  pointer-events: none;
}

/* 交互触发 */
.card-bubble:hover .breathing-ripple {
  opacity: 1;
}
```

### 11.2 对比度检查器 (Contrast Checker)

实时检测配色对比度，确保满足 WCAG 2.1 标准。

```typescript
interface ContrastChecker {
  check: (foregroundColor: string, backgroundColor: string) => {
    const luminance = this.getLuminance(backgroundColor);
    const textLuminance = this.getLuminance(foregroundColor);
    return luminance > textLuminance ? luminance : null;
  },
  
  getLuminance: (color: string) => {
    const rgb = this.hexToRgb(color);
    return 0.299 * rgb.r + 0.587 * rgb.g + 0.114 * rgb.b;
  },
  
  hexToRgb: (hex: string) => {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})$/.exec(hex);
    return result ? {
      r: parseInt(result[1], 16),
      g: parseInt(result[2], 16),
      b: parseInt(result[3], 16)
    } : { r: 0, g: 0, b: 0 };
  }
};

// UI 指示器
const ContrastIndicator = {
  good: { icon: 'check-circle', color: '#10B981', text: '对比度合格' },
  warning: { icon: 'warning', color: '#F59E0B', text: '对比度偏低' },
  fail: { icon: 'times-circle', color: '#EF4444', text: '不满足 WCAG 2.1' }
};
```

### 11.3 色弱模拟器 (Color Blindness Simulator)

模拟不同光照条件下的显示效果，验证色盲用户友好度。

```typescript
interface ColorBlindnessSimulator {
  simulate: (baseColor: string, condition: 'protanopia' | 'deuteranopia' | 'tritanopia') => {
    return this.adjustColor(baseColor, condition);
  },
  
  adjustColor: (color: string, condition: string) => {
    // 根据 ADR-007 (色盲适配) 的规则调整颜色
    // protanopia: 红色感知困难
    // deuteranopia: 绿色感知困难
    // tritanopia: 蓝色感知困难
    switch (condition) {
      case 'protanopia':
        // 提高红色成分
        return this.shiftHue(color, 30);
      case 'deuteranopia':
        // 提高绿色成分
        return this.shiftHue(color, -30);
      case 'tritanopia':
        // 调整明度/饱和度
        return color;
    }
  },
  
  shiftHue: (color: string, degrees: number) => {
    const hsl = this.hexToHsl(color);
    return this.hslToHex({
      h: (hsl.h + degrees) % 360,
      s: hsl.s,
      l: Math.max(20, Math.min(80, hsl.l)),
      a: hsl.a
    });
  }
};
```

### 11.4 信任锚点 (Trust Anchors - 详细设计)

数字签名印章的视觉锚点设计详情。

```css
/* 信任锚点容器 */
.trust-anchor {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 16px;
  height: 16px;
  
  /* 微小的数字指纹图标 */
  background: 
    linear-gradient(135deg, rgba(244, 230, 160, 0.1) 0%, rgba(244, 230, 160, 0.05) 20%);
  
  border-radius: 2px;
  border: 1px solid rgba(59, 130, 246, 0.2);
  
  font-size: 8px;
  line-height: 1;
  color: #6B7280;
  font-weight: bold;
}

/* 悬停提示 */
.trust-anchor:hover {
  cursor: help;
  transform: scale(1.2);
  transition: transform 0.2s ease;
}

/* 工具提示 */
.trust-anchor::before {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 120%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.9);
  color: #6B7280;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 10px;
  white-space: nowrap;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.trust-anchor:hover::before {
  opacity: 1;
}
```

### 11.5 播放/暂停滑块 (Playback Slider)

时光机的时间轴拖拽滑块控制。

```typescript
interface TimelineSliderProps {
  minTime: string;      // 最小时间
  maxTime: string;      // 最大时间
  currentTime: string;  // 当前时间
  isPlaying: boolean;    // 是否播放中
  onSeek: (time: string) => void;
  onPlayPause: () => void;
}

const TimelineSlider = {
  render: (props: TimelineSliderProps) => {
    const { minTime, maxTime, currentTime, isPlaying, onSeek, onPlayPause } = props;
    
    // 计算进度百分比
    const progress = this.calculateProgress(minTime, maxTime, currentTime);
    
    return (
      <div className="timeline-controls">
        <button onClick={onPlayPause}>
          {isPlaying ? '⏸' : '▶️'}
        </button>
        <input 
          type="range"
          min={minTime}
          max={maxTime}
          value={currentTime}
          onChange={onSeek}
          step="1000"
        />
        <div className="time-display">{this.formatTime(currentTime)}</div>
        <div className="progress-bar">
          <div style={{ width: `${progress}%` }} />
        </div>
      </div>
    );
  },
  
  calculateProgress: (min: string, max: string, current: string) => {
    const minMs = new Date(min).getTime();
    const maxMs = new Date(max).getTime();
    const currentMs = new Date(current).getTime();
    return ((currentMs - minMs) / (maxMs - minMs)) * 100;
  },
  
  formatTime: (time: string) => {
    return new Date(time).toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' });
  }
};
```

### 11.6 高对比度模式

为强光环境或视觉障碍用户设计的高对比度切换模式。

```css
/* 高对比度模式 */
.high-contrast-mode {
  /* 强制所有文字使用高对比度颜色 */
  --text-primary: #000000 !important;
  --text-secondary: #1F2937 !important;
  
  /* 卡片背景使用高对比度 */
  --card-bg: #FFFFFF !important;
  --card-border: #000000 !important;
  
  /* 移除所有渐变和半透明效果 */
  * {
    background: transparent !important;
    opacity: 1 !important;
  }
  
  /* 按钮边框加深 */
  .button-primary {
    border: 2px solid #000000 !important;
    box-shadow: none !important;
  }
}
```

### 11.7 安全沙箱 (Security Sandbox)

主题预览在隔离沙箱环境中运行，防止恶意 CSS 注入。

```typescript
// 沙箱组件
interface ThemePreviewSandbox {
  themeConfig: ThemeConfig;
  onApply: (config: ThemeConfig) => void;
  onCancel: () => void;
}

const ThemePreviewSandbox = {
  // CSP 配置
  sandboxStyle: {
    sandbox: 'allow-scripts',
    'sandbox-allow-popups': false,
    'sandbox-allow-same-origin': 'https://preview.hos.com'
  },
  
  // 主题变量白名单
  allowedTokens: [
    'primary-color', 'bg-app', 'border-radius', 
    'font-scale', 'logo-url'
  ],
  
  validate: (config: ThemeConfig) => {
    // 检查只包含允许的 token
    const configKeys = Object.keys(config);
    const invalidTokens = configKeys.filter(
      key => !this.allowedTokens.includes(key)
    );
    
    if (invalidTokens.length > 0) {
      throw new Error(`不允许的配置项: ${invalidTokens.join(', ')}`);
    }
  },
  
  // 实时对比度检测
  checkContrast: (primaryColor: string, textColor: string) => {
    const contrastRatio = this.calculateContrastRatio(primaryColor, textColor);
    if (contrastRatio < 4.5) {
      console.warn(`对比度不足: ${contrastRatio.toFixed(2)}`);
      return { valid: false, ratio: contrastRatio };
    }
    return { valid: true, ratio: contrastRatio };
  },
  
  calculateContrastRatio: (fg: string, bg: string) => {
    const fgLuminance = this.getLuminance(fg);
    const bgLuminance = this.getLuminance(bg);
    const lighter = Math.max(fgLuminance, bgLuminance);
    const darker = Math.min(fgLuminance, bgLuminance);
    return (lighter + 0.05) / (darker + 0.05);
  },
  
  getLuminance: (color: string) => {
    const rgb = this.hexToRgb(color);
    return 0.299 * rgb.r + 0.587 * rgb.g + 0.114 * rgb.b;
  }
};
```

### 11.8 无障碍降级策略 (Accessibility Fallback)

**遵循 WCAG 2.1**: 所有的动画和视觉特效必须支持 `prefers-reduced-motion` 和 `prefers-reduced-transparency` 媒体查询。

```css
/* 动画降级开关 */
@media (prefers-reduced-motion: reduce) {
  /* 禁用所有动画 */
  .breathing-ripple-container *,
  .notification-bubble[data-priority="critical"],
  .trust-anchor,
  .contrast-indicator {
    animation: none !important;
    transition: none !important;
    transform: none !important;
  }

  /* 磁力排斥动画降级为静态高亮 */
  .notification-bubble[data-priority="critical"] {
    border: 2px solid var(--danger-color) !important;
    background: #FEF2F2 !important;
  }

  /* 呼吸波纹降级为简单边框 */
  .breathing-ripple {
    display: none !important;
  }
}

/* 静默模式（用户手动关闭所有动效）*/
.silent-mode {
  .breathing-ripple-container *,
  .notification-bubble,
  .trust-anchor,
  .contrast-indicator {
    animation: none !important;
    transition: none !important;
    transform: none !important;
  }

  /* Critical 消息降级为红色边框 */
  .notification-bubble[data-priority="critical"] {
    border: 3px solid var(--danger-color) !important;
  }
}
```

```typescript
// 运行时检测用户偏好
const AccessibilityPreferences = {
  // 检测是否减少动画
  prefersReducedMotion: (): boolean => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  },

  // 检测是否减少透明度
  prefersReducedTransparency: (): boolean => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia('(prefers-reduced-transparency: reduce)').matches;
  },

  // 获取静默模式状态（本地存储）
  isSilentMode: (): boolean => {
    try {
      return localStorage.getItem('hos.silent_mode') === 'true';
    } catch {
      return false;
    }
  },

  // 切换静默模式
  toggleSilentMode: (enabled: boolean) => {
    try {
      localStorage.setItem('hos.silent_mode', String(enabled));
      document.body.classList.toggle('silent-mode', enabled);
    } catch {
      console.error('Failed to toggle silent mode');
    }
  },

  // 自动适配：当检测到用户偏好时自动降级
  autoAdapt: () => {
    const prefersReduced = this.prefersReducedMotion();
    const isSilent = this.isSilentMode();

    if (prefersReduced || isSilent) {
      document.body.classList.add('reduced-motion');
      console.info('Accessibility mode active: reduced animations');
    } else {
      document.body.classList.remove('reduced-motion');
    }
  }
};

// 组件初始化时调用
useEffect(() => {
  AccessibilityPreferences.autoAdapt();
}, []);
```

#### 无障碍降级策略对照表

| 动效名称 | 降级方式 | 预期效果 |
|---------|---------|---------|
| 呼吸波纹 | 移除 animation，隐藏 `.breathing-ripple` | 静态卡片边框 |
| 磁力排斥动画 | 移除 animation，改为红色粗边框 | 高优先级消息静态高亮 |
| Trust Anchor 悬停动画 | 移除 transform scale | 静态徽章 |
| 通知气泡进入动画 | 移除 enter 动画 | 消息直接出现 |
| 光晕效果 | 移除 box-shadow | 纯色块显示 |
| 渐变背景 | 改为纯色背景 | 高对比度背景 |

```css
/* 降级模式统一样式 */
.reduced-motion .notification-bubble {
  transition: none !important;
  animation: none !important;
}

.reduced-motion .notification-bubble[data-priority="critical"] {
  border: 3px solid #EF4444;
  background: #FEF2F2;
  padding: var(--spacing-3) var(--spacing-4);
}
```

