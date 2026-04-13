# ADR-001: 前端技术选型 (Frontend Technology Selection)

## 1. 状态 (Status)
已接受 (Accepted) - 2026-04-13

## 2. 上下文 (Context)
项目为一个多租户医疗 SaaS 系统 (HOS)，包含复杂的电子病历 (EMR)、挂号预约、进销存管理等 B 端后台功能，以及患者侧的小程序。
视觉设计上需要支持多租户的品牌定制 (White-labeling)，要求能快速切换主色调、圆角、间距等。

## 3. 候选方案 (Options)

### 方案 A: React 18 + Ant Design Pro
*   **优点**: 交付速度极快，组件库生态成熟，医疗 B 端常用，AI 生成代码支持好。
*   **缺点**: 样式深度定制成本高（需要覆盖 Less 变量或运行时注入全局 CSS），Bundle 较大。

### 方案 B: Next.js 14+ + Tailwind CSS + shadcn/ui
*   **优点**: SSR/SSG 支持，设计灵活性极高，Tailwind 非常适合做多租户 CSS 变量注入，无头组件 (Headless UI) 方便深度定制医疗交互。
*   **缺点**: 学习曲线较陡，大部分组件需要从 shadcn/ui 引入并自行维护，复杂 Table/Form 封装量大。

## 4. 决策结果 (Decision)

### 4.1 MVP Phase 1
**选择: 方案 A (React 18 + Ant Design Pro) + Tailwind CSS (作为辅助样式库)**

*   **Rationale**:
    1. 医疗 SaaS 核心诉求是”稳”和”快”。AntD Pro 提供的一站式 Layout、Table、Form 极大减少了基础业务逻辑的编写时间。
    2. 组件库生态成熟，医疗 B 端常用，AI 生成代码支持好。
    3. 为了解决视觉灵活性，引入 Tailwind 用于页面布局和原子化样式定制。
    4. 主题定制通过 AntD 的 `ConfigProvider` 动态注入主题 Token，配合 Tailwind 的 CSS Variables 模式实现全站视觉一致性。

### 4.2 Phase 2 试点
**选择: 方案 B (Next.js 14+ + Tailwind CSS + shadcn/ui) 作为试点**

*   **试点目的**: 验证 SSR/SSG 对性能的影响，评估无头组件对多租户定制的适配性
*   **试点范围**: 选择一个非核心模块进行试点，如患者端预约页面
*   **试点条件**: MVP 稳定运行至少 3 个月后启动

## 5. 后果 (Consequences)

*   MVP 阶段需要规范 AntD 与 Tailwind 类名的共存命名约定 (Prefix)。
*   放弃 Next.js 的 App Router 架构，回归纯 SPA 模式，但依然保留多租户动态加载逻辑。
*   Phase 2 试点需要额外评估框架切换的迁移成本。
*   技术栈决策已统一到本 ADR，避免团队并行开发时分叉。

## 6. 视觉设计规范 (Visual Design Guidelines)

**评审人**: @gemini

### 6.1 设计变量映射
Tailwind CSS 辅助样式中，必须严格映射到设计系统规范中的 CSS 变量：

```css
:root {
  /* 猫咖视觉基调 - 温暖、阳光 */
  --bg-app: #FDF8F3;
  --primary-color: #F59E0B;
  --text-primary: #1F2937;

  /* 医疗蓝 - 专业、可靠 */
  --medical-blue: #3B82F6;
  --success-color: #10B981;
  --warning-color: #F59E0B;
  --danger-color: #EF4444;
}
```

### 6.2 Tailwind 配置扩展
```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        'cat-cafe-bg': 'var(--bg-app)',
        'cat-cafe-primary': 'var(--primary-color)',
        'medical-blue': 'var(--medical-blue)',
      }
    }
  }
}
```

## 7. 后续行动项

- [ ] 创建 `docs/design-system.md` 定义完整的视觉设计规范
- [ ] 实现 AntD 主题组件，支持设计系统 CSS 变量映射
- [ ] 建立组件 Storybook，确保视觉一致性
