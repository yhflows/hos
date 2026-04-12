# HOS 前端视觉与交互架构规范 (Frontend Vision & UX)

> **目标**: 为口腔医生提供”零摩擦”的生产力工具，为患者提供”有温度”的就诊体验。
>
> **技术基线**: 严格遵循 [ADR-001: 前端技术选型](../decisions/001-frontend-tech-selection.md)、[ADR-002: 医疗影像标准](../decisions/002-medical-imaging-standard.md)、[ADR-003: 多租户 UI 定制与安全模型](../decisions/003-tenant-ui-security.md)。

## 1. 技术栈基线 (Technical Stack Baseline)

> 本文件的 UX 设计规范基于 **ADR-001: 前端技术选型** 的决策结果。

| 平台 | 技术选型 | 说明 |
|------|---------|------|
| **Web 管理端 (B端)** | React 18 + Ant Design Pro + Tailwind CSS | AntD 用于核心组件，Tailwind 用于布局和原子化样式定制 |
| **状态同步** | TanStack Query v5 + push-service (WS) | 声明式数据获取，支持 WebSocket 触发的实时缓存失效 |
| **移动端 (C端)** | Taro 3.x (React) | 与 Web 端共享组件逻辑，编译为微信小程序 + H5 |
| **桌面端** | MVP 暂缓，未来考虑 Tauri 2.0 | 静态导出 Web 产物封装（需评估 SSR 能力取舍） |

## 2. 核心 UX 模式

### 2.1 租户主题引擎 (Tenant Theme Engine)
> **安全约束**: 遵循 [ADR-003: 多租户 UI 定制与安全模型](../decisions/003-tenant-ui-security.md) 的白名单注入规范。

- **实现**: 后端返回租户配置，前端在 Root Layout 通过 CSS Variables 注入。
- **动态性**: 无需重新编译，一秒切换诊所品牌色。
- **定制范围**: 仅支持全局 CSS 变量层面（配色、圆角、字体缩放），不直接允许自定义 CSS 代码块。

### 2.2 实时“无感”刷新 (Real-time Sync)
- **协议**: `push-service` -> WebSocket -> 发送 Entity Invalidation。
- **行为**: 前端静默刷新 React Query 缓存，实现排班表等数据的实时更新。

### 2.3 AI 辅助诊疗视觉层
- **原则**: AI 信息以 Overlay 形式“漂浮”在病历上方，不干扰主流程输入。

## 3. 牙位图 (Tooth Chart) 核心组件
- **数据规范**: 采用 FDI/Palmer 标识法的 Tooth-JSON。
- **渲染方案**: Web 端使用响应式 SVG，移动端使用 Canvas 以提升触摸精度。
