# HOS 前端视觉与交互架构规范 (Frontend Vision & UX)

> **目标**: 为口腔医生提供“零摩擦”的生产力工具，为患者提供“有温度”的就诊体验。

## 1. 技术栈选型 (The "Modern" Stack)

| 平台 | 技术选型 | 理由 |
|------|---------|------|
| **Web 管理端 (B端)** | Next.js 14+ (App Router) + Tailwind CSS | SSR 极致加载速度 + CSS 变量级主题定制 |
| **组件库** | shadcn/ui + Radix UI | 无头组件(Headless)，方便针对医疗场景深度定制交互 |
| **状态同步** | TanStack Query v5 + push-service (WS) | 声明式数据获取，支持 WebSocket 触发的实时缓存失效 |
| **移动端 (C端)** | uni-app (Vue 3) / Taro (React) | 微信生态深度集成，兼顾 H5 传播力 |
| **桌面端** | Tauri 2.0 | 比 Electron 更小的体积，直接封装 Web 产物，支持调用打印机/影像设备 |

## 2. 核心 UX 模式

### 2.1 租户主题引擎 (Tenant Theme Engine)
- **实现**: 后端返回租户配置，前端在 Root Layout 通过 CSS Variables 注入。
- **动态性**: 无需重新编译，一秒切换诊所品牌色。

### 2.2 实时“无感”刷新 (Real-time Sync)
- **协议**: `push-service` -> WebSocket -> 发送 Entity Invalidation。
- **行为**: 前端静默刷新 React Query 缓存，实现排班表等数据的实时更新。

### 2.3 AI 辅助诊疗视觉层
- **原则**: AI 信息以 Overlay 形式“漂浮”在病历上方，不干扰主流程输入。

## 3. 牙位图 (Tooth Chart) 核心组件
- **数据规范**: 采用 FDI/Palmer 标识法的 Tooth-JSON。
- **渲染方案**: Web 端使用响应式 SVG，移动端使用 Canvas 以提升触摸精度。
