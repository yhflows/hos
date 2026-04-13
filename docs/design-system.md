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
