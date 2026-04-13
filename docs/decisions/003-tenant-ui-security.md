# ADR-003: 多租户 UI 定制与安全模型 (Tenant UI Customization & Security)

## 1. 状态 (Status)
已接受 (Accepted) - 2026-04-13

**评审人**: @opus, @gemini, @codex

## 2. 上下文 (Context)
系统需要支持多诊所个性化视觉定制 (White-labeling)，允许自定义主色、品牌 LOGO 和部分样式属性。

## 3. 决策结果 (Decision)
**方案: 基于 CSS 变量的白名单注入模型**

*   **定制约束**: 仅支持全局 CSS 变量层面的定制（配色、圆角、字体缩放）。不直接允许自定义 CSS 代码块。
*   **配置存储**: 在 `tenants` 配置表中增加 `theme_config` (JSONB)。
*   **运行时注入**: 
    1. 后端 SSR/BFF 根据 Tenant ID 加载配置。
    2. 生成 `:root { --primary-color: #xxx; ... }` 注入 HTML。
*   **安全验证 (关键)**:
    1. **Color Check**: 正则校验 `^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$`。
    2. **Dimension Check**: 校验 `rem`, `px` 单位，限制最大值。
    3. **Content Sanitization**: 拒绝任何包含 `javascript:`, `data:`, `eval` 的字符串。
*   **Asset Security**: LOGO 等素材必须上传至 MinIO 的租户隔离 Bucket，并经过 ImageMagick/Vips 实时校验和压缩。

## 4. 后果 (Consequences)
*   完全避免了 CSS 注入 (CSS Injection) 带来的 XSS 风险。
*   前端开发需严格遵守 CSS 变量驱动的样式编写规范。
*   不支持过于复杂的 UI 重排（布局定制需后续 Phase 讨论）。
