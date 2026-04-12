# HOS1-OPUS 系统架构设计文档

文档状态: Baseline
更新时间: 2026-04-04
适用范围: MVP Phase 1 到 Phase 2 的系统级设计基线
关联文档:

- `docs/modules/*.md`

---

## 1. 文档目标

本文档是 HOS1-OPUS 的系统级主设计文档，用于统一研发、测试、运维和架构评审的共同基线。

本文档只描述当前有效方案，不再保留历史废案。后续重大变更应通过 ADR 或新的评审文档追加，不在多个文档中并行维护“最终方案”。

---

## 2. 系统定位与范围

### 2.1 当前系统定位

HOS1-OPUS 当前定位为面向口腔医疗机构的多租户 SaaS 平台，核心目标是先跑通以下主链路：

- 身份认证与租户隔离
- 预约挂号与到诊流转
- 电子病历与影像管理
- 划价收费、退款与对账
- 实时通知与消息推送

宠物医疗、AI 分析、内容社区、商城广告等能力属于未来扩展方向，不进入当前主链路设计约束。

### 2.2 MVP Phase 1 范围

纳入 MVP 的模块：

- `IAM / Casdoor`
- `admin-bff / patient-bff / wechat-bff`
- `clinic-service`
- `booking-service`
- `emr-service`
- `payment-service`
- `push-service`
- `common/pkg`

明确暂缓或降级的能力：

- Odoo 实时库存锁定
- 多点执业无感切换
- AI 推理进入核心业务路径
- 商城、广告、论坛、IM 等边缘业务

### 2.3 非目标

以下内容不属于当前系统设计的硬约束范围：

- 将口腔与宠物两套业务流程在同一版本中同时完备建模
- 将 Odoo 作为在线收费链路的强依赖
- 以实时跨 Schema 查询支持总部报表
- 以 WebSocket 会话代替离线通知存储

---

## 3. 架构原则

- 单一真相源原则: 主设计文档和 ADR 才能承载当前有效结论。
- 核心链路本地闭环: 挂号、病历、收费必须在本域数据和本域事务中闭环。
- 同步最小化: 服务间默认少同步、多异步，避免链式故障。
- 合规优先: 审计、加密、销毁证明优先于“实现简单”。
- 多租户强隔离: 关系型数据、对象存储、缓存和通知通道都必须带租户边界。
- 重系统做旁路: Odoo、AI、报表等重系统默认在核心链路旁路接入。

---

## 4. 最终技术决策

| 决策点 | 最终方案 | 约束 |
|--------|---------|------|
| 外部入口 | Traefik 3.x | TLS 终结、路由分发 |
| 对外 API | BFF 层 REST/JSON | Web/小程序/App 统一接入 |
| 内部通信 | gRPC + Protobuf | 仅服务间使用 |
| 身份认证 | Casdoor API-only | 只承担身份认证与组织关系 |
| 业务令牌 | BFF 自签业务 JWT | 服务统一只验证业务 JWT |
| 关系数据库 | PostgreSQL 16+ | `Schema-per-tenant` |
| 缓存 | Redis 7+ | 会话、限流、短期缓存 |
| 事件总线 | NATS JetStream | Outbox 驱动的领域事件总线 |
| 实时推送 | push-service | 前端禁止直连 NATS |
| 对象存储 | MinIO | Bucket-per-tenant |
| 进销存 | Odoo 17+ + supplychain-bff | 防腐层隔离，MVP 不做在线强依赖 |
| 链路追踪 | OpenTelemetry + Jaeger | 全链路可观测 |
| 监控 | Prometheus + Grafana | 分级告警 |

---

## 5. 逻辑架构

### 5.1 分层结构

```text
Clients
├── Web Admin
├── WeChat Mini Program / H5
└── Mobile App

Edge
└── Traefik

Access Layer
├── admin-bff
├── patient-bff
├── wechat-bff
└── push-service

Core Domain Services
├── clinic-service
├── booking-service
├── emr-service
└── payment-service

Integration Layer
└── supplychain-bff -> Odoo

Infrastructure
├── PostgreSQL
├── Redis
├── NATS JetStream
├── MinIO
└── Casdoor
```

### 5.2 组件职责

| 组件 | 职责边界 |
|------|---------|
| `Traefik` | 外部入口、TLS 终结、基础路由 |
| `BFF` | 登录代理、业务 JWT 签发、聚合接口、端侧协议适配 |
| `push-service` | WebSocket 连接承接、业务 JWT 校验、通知下发 |
| `clinic-service` | 诊所配置、排班、医生与科室基础数据 |
| `booking-service` | 预约、签到、就诊状态机 |
| `emr-service` | 病历、牙位图、影像元数据 (遵循 ADR-002)、预签名 URL |
| `payment-service` | 账单、交易、退款、回调幂等 |
| `supplychain-bff` | Odoo 防腐层、租户到 company 映射、异步对账 |
| `PostgreSQL` | 核心事务数据、租户路由与公共元数据 |
| `NATS JetStream` | 领域事件与集成事件 |
| `MinIO` | 影像和附件对象存储 |
| `Casdoor` | 身份认证、用户和组织关系 |

### 5.3 控制平面职责

租户开通、冻结、切库、迁移、退租和销毁证明属于控制平面职责。

MVP 阶段可以由 `admin-bff + internal jobs` 实现该编排；当租户规模和运维复杂度提升后，再独立为专门服务。业务服务本身不直接承担租户生命周期编排。

---

## 6. 认证、授权与租户上下文

### 6.1 双令牌模型

- Casdoor 负责认证用户身份
- BFF 负责选择当前租户并签发业务 JWT
- 业务服务和 `push-service` 统一只验证业务 JWT

正常业务流中，微服务不直接依赖 Casdoor Token 作为业务访问令牌。

### 6.2 登录流程

1. 前端将账号密码、短信验证码或微信登录 Code 提交给对应 BFF。
2. BFF 作为后端代理调用 Casdoor 登录接口。
3. BFF 根据用户的组织关系、角色关系和当前租户选择结果，签发业务 JWT。
4. 前端后续所有业务 API 和 WebSocket 握手都只携带业务 JWT。

建议的令牌策略：

- Access Token: 15 分钟
- Refresh Token: 7 天
- 业务签名密钥与 Casdoor 密钥分离
- 平台通过 JWKS 或公钥分发业务 JWT 的验签公钥

### 6.3 业务 JWT 声明

```json
{
  "sub": "user_12345",
  "tenant_id": "tenant_999",
  "roles": ["doctor"],
  "scope": ["booking:read", "emr:write"],
  "token_version": 1,
  "exp": 1711680000
}
```

### 6.4 多点执业策略

多点执业无感切换不属于 MVP 范围。当前版本只允许一个业务 JWT 对应一个当前租户。

后续若支持多租户切换，仍然采用“切换后签发新业务 JWT”的方式，不在业务请求中携带多个活跃租户上下文。

---

## 7. 多租户隔离与数据存储

### 7.1 PostgreSQL 隔离模型

数据库采用共享 PostgreSQL 集群 + `Schema-per-tenant` 模型：

- `public` schema 存放平台公共元数据
- `tenant_{id}` schema 存放租户业务数据

`public` 中至少包含以下表：

- `tenants`
- `tenant_db_routes`
- `tenant_odoo_mapping`
    *   **策略实现**：
        1.  **数据库Schema隔离**：核心业务数据采用“Database per service, Schema per tenant”的模式。为每个新入驻的医疗机构在 PostgreSQL 中开辟独立的 Schema。这种物理到逻辑的一体化隔离能确保在进行数据备份、迁移、分析或清除时，“一斩即断”，绝不会误碰其他诊所的医患数据。
        2.  **强制拦截层 (GORM Hooks)**：在后端框架层，所有对微服务 API 的调用必须先经过 JWT Auth 中间件解析出 `TenantID`。随后的 GORM DB 实例在启动事务前自动执行 `SET LOCAL search_path TO ?`，在底座级彻底封杀因代码 Bug 导致“越权查看他人病历”的可能性。

### 7.2 租户访问红线
        3.  **UI 个性化安全 (ADR-003)**: 多租户主题定制仅限 CSS 变量白名单，严禁 CSS 注入，确保 White-labeling 的视觉表现与系统安全并重。

所有租户绑定的读写都必须遵守以下规则：

1. 先从上下文中解析 `tenant_id`
2. 通过路由表获取目标 DB 集群
3. 在事务内执行 `SET LOCAL search_path TO tenant_{id}, public`
4. 在同一个事务闭环内完成后续查询和写入

禁止事项：

- 在事务外执行 `SET LOCAL`
- 绕过统一封装直接手写跨租户 SQL
- 使用可能跨 `search_path` 复用的连接级缓存而不做隔离验证

### 7.3 扩展策略

`Schema-per-tenant` 是 MVP 的主策略，但必须预留扩容出口：

- 中小租户共享集群
- 大租户可切换至独立数据库集群
- 私有化客户可独立部署

平台通过 `tenant_db_routes` 做 DB Router，不把租户与单一集群硬绑定在代码里。

### 7.4 报表策略

总部或跨租户统计不走实时跨 Schema 联查。统一采用异步投影和预聚合：

- 核心业务库承担 OLTP
- `report` 投影承担聚合和大查询

---

## 8. 对象存储与敏感数据保护

### 8.1 MinIO 隔离策略

- 每个租户独立 Bucket: `t-{tenant_id}`
- 上传和读取都通过后端签发的 Presigned URL
- 大文件采用 Multipart Upload

### 8.2 影像访问原则

- 前端不保存永久可访问的对象直链
- 每次访问都经过业务鉴权并签发短时效 URL
- 遵循 ADR-002 (DICOM 标准)，影像元数据按 FHIR ImagingStudy 存入数据库，二进制对象存入 MinIO

### 8.3 敏感信息处理

- 患者实名、手机号、证件等字段使用 AES-GCM 做物理加密
- BFF 对外返回时做脱敏展示
- 审计日志记录访问行为，不落裸敏感值

---

## 9. 事件驱动、推送与最终一致性

### 9.1 Outbox 规范

所有跨服务事件必须使用 Outbox 模式：

1. 业务数据变更与 Outbox 记录在同一事务提交
2. Relay 使用 `FOR UPDATE SKIP LOCKED` 扫描待发送记录
3. 发布成功后更新状态
4. 消费端按消息 ID 做幂等

禁止直接在业务代码中调用裸 `nats.Publish()`。

### 9.2 NATS JetStream 的职责边界

JetStream 只承载以下两类消息：

- 领域事件
- 对外系统集成事件

JetStream 不作为前端可任意订阅的消息系统，也不承载未经收口的 WebSocket 主题设计。

### 9.3 push-service 设计约束

`push-service` 是前端实时通信的唯一受信任入口。

允许订阅的主题必须被平台收口为有限集合，例如：

- `t.{tenant_id}.notify.user.{user_id}`
- `t.{tenant_id}.notify.role.{role}`
- `t.{tenant_id}.notify.broadcast`

不允许前端传入任意 Subject 进行动态订阅。

### 9.4 通知与领域事件解耦

领域事件不直接等价于前端通知。若某个业务事件需要推送：

1. 领域服务先发布领域事件
2. 通知编排逻辑将其转换为通知主题或通知投影
3. `push-service` 只负责向在线连接投递
4. 离线未读由通知中心或收件箱机制承载

---

## 10. 供应链与 Odoo 集成

### 10.1 设计原则

Odoo 是重要集成系统，但不是核心医疗主链路的在线强依赖。

### 10.2 MVP 约束

MVP 阶段：

- 价格目录应在本域维护快照或投影
- 核心支付链路不等待 Odoo 在线响应
- Odoo 实时库存锁定暂缓
- 支付成功后的库存扣减、进销存记账和对账采用异步处理

### 10.3 supplychain-bff 职责

- 维护 `tenant_id -> odoo_company_id` 映射
- 负责 Odoo RPC 适配、错误翻译、限流与重试
- 承担租户开通和退租阶段的 Odoo 编排

任何服务都不得直接写 Odoo 底层数据库。

---

## 11. 核心运行时流程

### 11.1 预约到收费主链路

1. 患者发起预约，`booking-service` 创建预约单。
2. 到诊签到后进入就诊状态流转。
3. 医生在 `emr-service` 内完成病历与牙位图录入。
4. 就诊结束后，`payment-service` 生成账单并处理支付。
5. 支付结果通过 Outbox 发布 `payment.completed` 事件。
6. 通知服务或推送编排将结果转换为用户通知。
7. `push-service` 向患者端或前台端推送状态变化。
8. 供应链和账务系统异步消费事件，执行 Odoo 对账或库存动作。

### 11.2 影像上传链路

1. 前端请求上传权限。
2. `emr-service` 校验权限后签发短时效 Presigned URL。
3. 前端直传 MinIO。
4. 上传完成后回写对象元数据和业务关联。

### 11.3 退租与销毁链路

退租流程不是“收到申请立即删除”，而是以下状态机：

`active -> suspended -> pending_erasure -> erased`

执行步骤：

1. 冻结登录、写入和计费入口
2. 导出销毁摘要与审批凭据
3. 擦除 PostgreSQL 租户 Schema
4. 清理 MinIO 对象和 Bucket
5. 调用 Odoo 定制清理接口完成业务数据擦除与 company 归档
6. 清理 Redis 缓存和 NATS 对应主题数据
7. 生成销毁证明，审计材料保留 7 年

---

## 12. 可观测性与运维基线

### 12.1 目标

- RPO < 1 分钟
- RTO < 15 分钟
- 支付、Outbox、WebSocket 连接和租户限流必须具备独立监控

### 12.2 必备告警

- P0: 支付失败率 > 3%
- P1: Outbox Pending 积压 > 200
- P1: Push 连接异常断崖下降
- P1: NATS DLQ 或死信主题持续堆积
- P1: 单租户 QPS 长时间打满限流阈值

### 12.3 发布策略

- 开发环境使用 Docker Compose
- 生产环境使用 K8s
- 默认 Rolling Update，大特性支持蓝绿或灰度

### 12.4 性能基准

MVP 结束后必须完成：

- `k6` 网关和核心服务压测
- 500 个 Schema 并发查询退化测试
- Outbox 大量积压与恢复演练

---

## 13. 工程红线

- 禁止在非事务上下文执行租户数据访问
- 禁止直接调用裸 `nats.Publish`
- 禁止绕过 `supplychain-bff` 直接操作 Odoo
- 禁止在非 `push-service` 服务中暴露 WebSocket 入口
- 禁止前端自行指定 NATS Subject
- 禁止在用户退租时跳过冻结和审计流程直接销毁数据

---

## 14. 结论

HOS1-OPUS 的系统基座确定为：

- Go-Zero 微服务 + BFF
- Casdoor API-only + 业务 JWT
- PostgreSQL `Schema-per-tenant`
- Outbox + NATS JetStream
- push-service 统一实时推送入口
- supplychain-bff 隔离 Odoo

在此基线下，MVP 应优先完成 Booking、EMR、Payment、Push 四条核心链路，并把高复杂度外部系统控制在异步集成边界之外。
