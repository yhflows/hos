# ADR-001: 前端技术选型 (Frontend Technology Selection)

## 1. 状态 (Status)
已接受 (Accepted) - 2026-04-12

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
**倾向选择: 方案 A (React + Ant Design Pro) + Tailwind CSS (作为辅助样式库)**

*   **Rationale**: 
    1. 医疗 SaaS 核心诉求是“稳”和“快”。AntD Pro 提供的一站式 Layout、Table、Form 极大减少了基础业务逻辑的编写时间。
    2. 为了解决视觉灵活性，引入 Tailwind 用于页面布局和原子化样式定制，不直接依赖 AntD 的默认样式隔离。
    3. 主题定制通过 AntD 的 `ConfigProvider` 动态注入主题 Token，配合 Tailwind 的 CSS Variables 模式实现全站视觉一致性。

## 5. 后果 (Consequences)
*   需要规范 AntD 与 Tailwind 类名的共存命名约定 (Prefix)。
*   放弃 Next.js 的 App Router 架构，回归纯 SPA 模式，但依然保留多租户动态加载逻辑。
