# emr-service 详细设计草案

> 状态: 草案 v3 - 已纳入全部团队评审意见
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`emr-service` 负责 HOS 系统的电子病历 (EMR) 管理、牙位图数据结构、医疗影像元数据索引，是核心诊疗数据的唯一真相源。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 电子病历创建、修改、查询 | ✅ | - |
| 牙位图数据结构化存储 | ✅ | - |
| 影像元数据索引 (遵循 ADR-002) | ✅ | 影像二进制存储 (MinIO) |
| 就诊记录管理 | ✅ | - |
| 诊断与处方管理 | ✅ | - |
| DICOM 文件解析 | ✅ | - |
| 医生/科室/排班基础数据 | ❌ | clinic-service |
| 影像查看器渲染 | ❌ | 前端 (CornerstoneJS/OHIF) |
| PACS 服务器 | ❌ | Phase 2 引入 Orthanc |

### 1.2 核心约束

- **多租户隔离**: 所有操作必须在租户 schema 内执行
- **医疗合规**: 病历数据必须完整保留，不可删除，只允许修正
- **病历快照不可变性**: 签名后病历不可修改，只能创建新 revision
- **事务边界一致性**: encounters 是事务边界，所有子表操作须挂同一 revision_token
- **审计日志**: 病历修改必须记录完整变更历史
- **最终一致性**: 跨服务事件通过 Outbox 模式发布
- **DICOM 标准**: 遵循 ADR-002 医疗影像标准
- **AI 结果独立**: AI 发现与医生诊断分层存储，不混淆

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### encounters (就诊记录表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| patient_id | VARCHAR | 患者 ID | IDX |
| clinic_id | VARCHAR | 诊所 ID | IDX |
| doctor_id | VARCHAR | 医生 ID | IDX |
| department_id | VARCHAR | 科室 ID | IDX |
| appointment_id | VARCHAR | 预约 ID | FK, IDX |
| encounter_type | VARCHAR | 就诊类型 (初诊/复诊/急诊/复查) | - |
| visit_type | VARCHAR | 访问类型 (门诊/急诊/随访) | - |
| status | VARCHAR | 状态 (in_progress/completed/cancelled) | IDX |
| current_revision_token | VARCHAR | 当前修订令牌 (事务边界) | IDX |
| is_signed | BOOLEAN | 是否已签名 | IDX |
| signed_by | VARCHAR | 签名人 ID | IDX |
| signed_at | TIMESTAMP | 签名时间 | IDX |
| signature_hash | VARCHAR | 签名哈希 (防篡改) | - |
| start_time | TIMESTAMP | 就诊开始时间 | - |
| end_time | TIMESTAMP | 就诊结束时间 | - |
| chief_complaint | TEXT | 主诉 | - |
| present_illness | TEXT | 现病史 | - |
| past_history | TEXT | 既往史 | - |
| allergy_history | TEXT | 过敏史 | - |
| notes | TEXT | 医生备注 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | - |

**索引策略**:
- `(tenant_id, patient_id, created_at DESC)` - 查询患者就诊历史
- `(tenant_id, doctor_id, start_time DESC)` - 查询医生诊疗记录
- `(tenant_id, current_revision_token)` - 事务边界查询

**状态约束**:
- 签名后 (`is_signed = true`) 禁止直接修改，必须创建新 revision
- 事务边界: 所有子表操作必须携带 `current_revision_token`

#### encounter_revisions (病历快照表，支持不可变签名)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (唯一标识) | UNIQUE, IDX |
| revision_number | INT | 修订序号 | IDX |
| snapshot_hash | VARCHAR | 快照哈希 (SHA-256) | - |
| is_signed | BOOLEAN | 是否已签名 | IDX |
| signed_by | VARCHAR | 签名人 ID | IDX |
| signed_at | TIMESTAMP | 签名时间 | IDX |
| notes | TEXT | 修订说明 | - |
| snapshot_data | JSONB | 完整快照数据 (压缩) | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**快照数据结构**:
```json
{
  "revision_token": "rev_abc123",
  "snapshot_hash": "a1b2c3d4...",
  "encounter": {...},
  "clinical_notes": [{...}, {...}],
  "diagnoses": [{...}],
  "treatments": [{...}],
  "prescriptions": [{...}],
  "tooth_chart": {...},
  "ai_findings": [{...}]
}
```

**签名流程约束**:
1. 创建 encounter 时自动生成 `revision_token`
2. 所有子表操作必须携带相同 `revision_token`
3. 签名时生成 `snapshot_hash`，设置 `is_signed = true`
4. 签名后修改必须创建新 revision，不能在原 revision 上继续操作

#### clinical_notes (临床笔记表，支持多次修正)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (事务边界) | IDX |
| note_type | VARCHAR | 笔记类型 (主观检查/客观检查/评估/计划) | - |
| content | TEXT | 笔记内容 | - |
| author_id | VARCHAR | 作者 ID | IDX |
| version | INT | 版本号 | - |
| is_latest | BOOLEAN | 是否最新版本 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**事务约束**:
- 所有写操作必须携带 `revision_token`
- 同一 `revision_token` 下的版本号连续递增

#### diagnoses (诊断表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (事务边界) | IDX |
| diagnosis_code | VARCHAR | 诊断代码 (ICD-10/ICD-11) | IDX |
| diagnosis_name | TEXT | 诊断名称 | - |
| diagnosis_type | VARCHAR | 诊断类型 (主要/次要/疑似) | - |
| certainty | VARCHAR | 确定性 (确诊/疑似/排除) | - |
| created_at | TIMESTAMP | 创建时间 | - |

#### treatments (治疗表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (事务边界) | IDX |
| treatment_code | VARCHAR | 治疗代码 | IDX |
| treatment_name | TEXT | 治疗名称 | - |
| tooth_number | VARCHAR | 牙位编号 (FDI: 18, 28, etc.) | IDX |
| treatment_type | VARCHAR | 治疗类型 (充填/根管/拔牙/修复) | - |
| status | VARCHAR | 状态 (planned/in_progress/completed/cancelled) | IDX |
| notes | TEXT | 治疗备注 | - |
| created_at | TIMESTAMP | 创建时间 | - |

#### prescriptions (处方表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (事务边界) | IDX |
| drug_name | TEXT | 药品名称 | - |
| drug_code | VARCHAR | 药品编码 | IDX |
| dosage | VARCHAR | 用法用量 | - |
| frequency | VARCHAR | 用药频率 | - |
| duration | VARCHAR | 用药时长 | - |
| route | VARCHAR | 给药途径 (口服/注射/外用) | - |
| created_at | TIMESTAMP | 创建时间 | - |

#### tooth_charts (牙位图表，遵循 FDI/Palmer 标准)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| patient_id | VARCHAR | 患者 ID | IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| revision_token | VARCHAR | 修订令牌 (事务边界) | IDX |
| chart_data | JSONB | 牙位图完整数据 (结构化) | - |
| notation | VARCHAR | 记法类型 (FDI/ISO/Palmer) | - |
| created_at | TIMESTAMP | 创建时间 | - |
| updated_at | TIMESTAMP | 更新时间 | - |

**chart_data 结构 (FDI 记法，支持图层和跨牙关联)**:
```json
{
  "layers": {
    "anatomy": {},  // 解剖层 (基础结构，不可变)
    "pathology": {},  // 病理层 (龋齿、缺失等)
    "treatment": {}  // 治疗层 (冠、桥、植入)
  },
  "upper_right": {
    "18": {
      "anatomy": {"status": "healthy"},
      "pathology": {"status": "healthy"},
      "treatment": {"notes": ""}
    },
    "17": {
      "anatomy": {"status": "healthy"},
      "pathology": {"status": "decayed", "notes": "浅龋", "depth": "superficial"},
      "treatment": {"notes": ""}
    },
    "16": {
      "anatomy": {"status": "healthy"},
      "pathology": {"status": "filled", "notes": "树脂充填"},
      "treatment": {"material": "composite", "notes": ""}
    },
    "15": {
      "anatomy": {"status": "missing"},
      "pathology": {"status": "missing", "notes": "已拔除"},
      "treatment": {"notes": ""}
    },
    "14": {
      "anatomy": {"status": "healthy"},
      "pathology": {"status": "healthy"},
      "treatment": {"notes": ""}
    }
  },
  // ... 其他象限
  "bridges": [
    {
      "id": "bridge_001",
      "type": "fixed",
      "teeth": [14, 15, 16],  // 跨牙关联
      "material": "zirconia",
      "notes": "三单位固定桥"
    }
  ]
}
```

**牙位状态枚举**:
| 状态 | 说明 |
|------|------|
| healthy | 健康 |
| decayed | 龋坏 |
| filled | 充填 |
| missing | 缺失 |
| root_treated | 根管治疗 |
| crown | 冠修复 |
| bridge | 桥修复 |
| implant | 种植 |
| denture | 义齿 |
| mobility | 松动 |
| fracture | 折裂 |

#### imaging_studies (影像研究表，遵循 ADR-002 DICOM/FHIR 标准)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| patient_id | VARCHAR | 患者 ID | IDX |
| encounter_id | VARCHAR | 就诊记录 ID | FK, IDX |
| study_instance_uid | VARCHAR | DICOM Study Instance UID (唯一) | UNIQUE, IDX |
| upload_session_id | VARCHAR | 上传会话 ID (幂等) | IDX |
| is_finalized | BOOLEAN | 是否已完成上传和解析 | IDX |
| finalized_at | TIMESTAMP | 完成时间 | IDX |
| study_date | DATE | 影像日期 | IDX |
| modality | VARCHAR | 影像模态 (CR/CT/MG/PT/XA) | IDX |
| body_part | VARCHAR | 检查部位 | - |
| study_description | TEXT | 影像描述 | - |
| series_count | INT | 序列数量 | - |
| instance_count | INT | 实例数量 | - |
| storage_path | VARCHAR | MinIO 存储路径 | - |
| thumbnail_url | VARCHAR | 缩略图 URL (自动生成 WebP) | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引约束**:
- `UNIQUE(tenant_id, study_instance_uid)` - 租户内 Study UID 唯一
- `UNIQUE(tenant_id, upload_session_id)` - 上传会话幂等

**DICOM 三步上传流程**:
1. **upload_session**: 客户端发起上传，生成 session_id
2. **object finalize**: 所有实例上传完成后，客户端确认，触发 DICOM 解析
3. **async parse/index**: 异步解析 DICOM 标签，建立索引
4. **thumbnail generation**: 自动生成 WebP 缩略图

**幂等与完整性约束**:
- 上传会话幂等: 同一 session_id 重复确认直接返回已完成状态
- UID 唯一约束: 租户内 study_uid/series_uid/sop_instance_uid 必须唯一
- 完整性校验: 实例数量与声明不符时拒绝 finalize

#### ai_findings (AI 发现表，独立于医生诊断)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| study_id | VARCHAR | 影像研究 ID | FK, IDX |
| series_id | VARCHAR | 序列 ID | FK, IDX |
| source_instance_uid | VARCHAR | 来源实例 UID | IDX |
| model_id | VARCHAR | AI 模型 ID | IDX |
| model_version | VARCHAR | 模型版本 | - |
| finding_type | VARCHAR | 发现类型 (caries/periodontitis/fracture) | IDX |
| confidence | DECIMAL | 置信度 (0-1) | - |
| bounding_box | JSONB | 边界框坐标 [x,y,width,height] | - |
| description | TEXT | AI 描述 | - |
| review_status | VARCHAR | 审核状态 (pending/accepted/rejected) | IDX |
| reviewed_by | VARCHAR | 审核人 ID | IDX |
| reviewed_at | TIMESTAMP | 审核时间 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**AI 结果存储原则**:
- AI 发现独立存储，不与医生诊断混淆
- 需医生审核后才关联到诊断表
- 审核状态可追溯
- 支持多模型并行分析

#### imaging_series (影像序列表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| study_id | VARCHAR | 影像研究 ID | FK, IDX |
| series_instance_uid | VARCHAR | DICOM Series Instance UID | UNIQUE, IDX |
| series_number | INT | 序列号 | - |
| modality | VARCHAR | 影像模态 | - |
| body_part | VARCHAR | 检查部位 | - |
| series_description | TEXT | 序列描述 | - |
| instance_count | INT | 实例数量 | - |
| created_at | TIMESTAMP | 创建时间 | - |

#### outbox_events (Outbox 模式)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | IDX |
| aggregate_type | VARCHAR | 聚合类型 (encounter, imaging_study) | IDX |
| aggregate_id | VARCHAR | 聚合 ID | IDX |
| event_type | VARCHAR | 事件类型 | - |
| event_data | JSONB | 事件数据 | - |
| published | BOOLEAN | 是否已发布 | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

---

## 3. API 接口设计

### 3.1 就诊记录相关

#### POST /api/v1/encounters

创建就诊记录（由 appointment.started 事件触发）

**请求**:
```json
{
  "appointment_id": "apt_abc123",
  "encounter_type": "初诊",
  "visit_type": "门诊",
  "chief_complaint": "牙痛持续3天",
  "present_illness": "右下颌磨牙区域疼痛，冷热刺激敏感",
  "past_history": "高血压3年，控制良好",
  "allergy_history": "青霉素过敏"
}
```

**响应**:
```json
{
  "id": "enc_123",
  "status": "in_progress",
  "start_time": "2026-04-15T09:12:00Z",
  "patient_id": "patient_123",
  "doctor_id": "doctor_123"
}
```

#### GET /api/v1/encounters/:id

获取就诊记录详情

#### PUT /api/v1/encounters/:id

更新就诊记录

#### POST /api/v1/encounters/:id/complete

完成就诊

**请求**:
```json
{
  "end_time": "2026-04-15T09:45:00Z",
  "summary": "常规检查，建议定期复查"
}
```

### 3.2 临床笔记相关

#### POST /api/v1/encounters/:id/notes

添加临床笔记

**请求**:
```json
{
  "note_type": "subjective",
  "content": "患者自述疼痛评分 7/10，夜间加重"
}
```

#### GET /api/v1/encounters/:id/notes

获取就诊笔记历史

### 3.3 诊断相关

#### POST /api/v1/encounters/:id/diagnoses

添加诊断

**请求**:
```json
{
  "diagnosis_code": "K02.1",
  "diagnosis_name": "牙釉质龋",
  "diagnosis_type": "主要",
  "certainty": "确诊"
}
```

### 3.4 治疗相关

#### POST /api/v1/encounters/:id/treatments

添加治疗

**请求**:
```json
{
  "treatment_code": "T001",
  "treatment_name": "树脂充填",
  "tooth_number": "46",
  "treatment_type": "充填",
  "notes": "浅龋，无需麻醉"
}
```

### 3.5 处方相关

#### POST /api/v1/encounters/:id/prescriptions

添加处方

**请求**:
```json
{
  "drug_name": "阿莫西林胶囊",
  "drug_code": "D001",
  "dosage": "0.5g",
  "frequency": "每日3次",
  "duration": "3天",
  "route": "口服"
}
```

### 3.6 牙位图相关

#### POST /api/v1/tooth-charts

创建/更新牙位图

**请求**:
```json
{
  "patient_id": "patient_123",
  "encounter_id": "enc_123",
  "notation": "FDI",
  "chart_data": {
    "upper_right": {
      "46": {"status": "decayed", "notes": "浅龋"}
    }
  }
}
```

#### GET /api/v1/patients/:patient_id/tooth-charts/latest

获取患者最新牙位图

#### GET /api/v1/tooth-charts/:id

获取牙位图详情

### 3.7 影像相关 (遵循 ADR-002)

#### POST /api/v1/imaging-studies

上传 DICOM 影像（分步上传）

**第1步：请求上传权限**
**响应**:
```json
{
  "study_id": "study_abc123",
  "upload_urls": [
    {"series_id": "series_001", "instance_id": "inst_001", "url": "..."},
    {"series_id": "series_001", "instance_id": "inst_002", "url": "..."}
  ],
  "presigned_urls_ttl": 3600
}
```

**第2步：前端直传 MinIO 后确认**
**请求**:
```json
{
  "study_id": "study_abc123",
  "uploaded_instances": ["inst_001", "inst_002"]
}
```

**第3步：解析 DICOM 并索引**
**响应**:
```json
{
  "study_id": "study_abc123",
  "study_instance_uid": "1.2.840.113619.2.55.3...",
  "modality": "CR",
  "series_count": 1,
  "instance_count": 2,
  "body_part": "右下颌磨牙"
}
```

#### GET /api/v1/imaging-studies/:id

获取影像研究详情

#### GET /api/v1/patients/:patient_id/imaging-studies

查询患者影像历史

---

## 4. 事件设计 (Outbox 模式)

### 4.1 领域事件

| 事件类型 | 触发时机 | 消费方 | 说明 |
|---------|---------|--------|------|
| `encounter.started` | 就诊记录创建 | push-service, clinic-service | 发送就诊开始通知 |
| `encounter.completed` | 就诊完成 | payment-service, push-service | 触发费用计算 |
| `diagnosis.created` | 诊断添加 | push-service, analytics | 更新患者健康档案 |
| `treatment.created` | 治疗添加 | push-service, supplychain-bff | 同步耗材消耗 |
| `imaging_study.uploaded` | 影像上传完成 | push-service, ai-service | 触发 AI 分析 |
| `imaging_study.ai_processed` | AI 分析完成 | push-service | 推送 AI 结果给医生 |

### 4.2 事件数据结构

```json
{
  "event_type": "imaging_study.uploaded",
  "aggregate_id": "study_abc123",
  "tenant_id": "tenant_999",
  "data": {
    "study_id": "study_abc123",
    "patient_id": "patient_123",
    "modality": "CR",
    "body_part": "右下颌磨牙",
    "instance_count": 2
  },
  "occurred_at": "2026-04-15T09:30:00Z"
}
```

---

## 5. DICOM 集成 (遵循 ADR-002)

### 5.1 文件处理流程

```
1. 前端选择 DICOM 文件
   ↓
2. BFF 请求上传权限
   ↓
3. emr-service 解析 DICOM 文件头 (读取 Study/Series/Instance UID)
   ↓
4. 生成 MinIO 预签名 URL
   ↓
5. 前端直传到 MinIO: t-{tenant_id}/dicom/{study_uid}/{series_uid}/{instance_uid}.dcm
   ↓
6. 前端确认上传完成
   ↓
7. emr-service 索引元数据到 imaging_studies / imaging_series 表
   ↓
8. 发布 imaging_study.uploaded 事件
```

### 5.2 DICOM 解析

使用 Go 库 `github.com/suyashkumar/dicom` 解析 DICOM 标签：

| DICOM Tag | 含义 | 存储位置 |
|-----------|------|----------|
| (0020,000D) Study Instance UID | 研究 UID | study_instance_uid |
| (0008,0060) Modality | 影像模态 | modality |
| (0018,0015) Body Part Examined | 检查部位 | body_part |
| (0008,1030) Study Description | 研究描述 | study_description |
| (0020,000E) Series Instance UID | 序列 UID | series_instance_uid |
| (0020,0011) Series Number | 序列号 | series_number |
| (0020,0013) Instance Number | 实例号 | instance_number |

### 5.3 Phase 2: PACS 集成

预留 Orthanc 服务器集成点，支持：
- DICOM Query/Retrieve (C-FIND/C-MOVE)
- HL7 ORM/OHR 消息路由
- 与医院现有 PACS 互联互通

---

## 6. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `clinic-service` | gRPC | 查询医生、科室、患者信息 |
| `payment-service` | 事件消费 | 就诊完成时触发费用计算 |
| `push-service` | gRPC | 推送就诊、影像通知 |
| `ai-service` (Phase 2) | 事件消费 | AI 影像分析 |
| `MinIO` | 直接连接 | DICOM 文件存储 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `NATS JetStream` | Outbox | 发布领域事件 |

---

## 7. 安全与合规

### 7.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 创建/修改就诊记录 | doctor, admin | `emr:write` |
| 查看就诊记录 | doctor, admin | `emr:read` |
| 添加诊断/治疗/处方 | doctor, admin | `emr:clinical_write` |
| 上传影像 | doctor, admin | `emr:imaging_upload` |
| 查看影像 | doctor, admin | `emr:imaging_read` |
| 患者查看自己病历 | patient | `emr:self_read` |

### 7.2 审计日志

记录以下关键操作：
- 就诊记录创建/修改/完成
- 临床笔记每次修改（带版本号）
- 诊断/治疗/处方添加
- 影像上传/查看
- 牙位图修改
- 操作人、时间、IP、修改前/后值

### 7.3 数据保留策略

- **就诊记录**: 永久保留，不可删除
- **临床笔记**: 永久保留所有版本
- **影像**: 根据《医疗病历管理规定》至少保存 30 年
- **删除权限**: 只有平台管理员在特定流程下可执行数据销毁

---

## 8. 性能指标

| 指标 | 目标值 |
|------|--------|
| 就诊记录查询响应时间 | < 200ms |
| 牙位图加载响应时间 | < 150ms |
| DICOM 文件上传响应时间 | < 500ms (首次解析) |
| 影像索引写入时间 | < 1s |
| 并发就诊记录处理能力 | > 500 TPS |
| 临床笔记版本查询 | < 100ms |

---

## 9. 待讨论事项

### 9.1 业务规则

1. **病历模板**: 是否支持预设病历模板？（建议：支持科室级模板）
2. **诊断编码体系**: 采用 ICD-10 还是 ICD-11？
3. **治疗编码体系**: 是否采用 CDT (Current Dental Terminology)？
4. **病历共享**: 跨诊所病历共享如何实现？（连锁场景）
5. **电子签名**: 病历完成是否需要医生电子签名？

### 9.2 已采纳的团队建议

| 来源 | 建议 | 状态 |
|------|------|------|
| @gemini | 牙位图图层渲染 + 跨牙关联 | ✅ 已纳入 (layers + bridges) |
| @gemini | 临床笔记语义化模板 + 版本对比视图 | ✅ 已纳入 (clinical_notes version control) |
| @gemini | AI RoI 叠加 + 胶片感交互 | ✅ 已纳入 (ai_findings bounding_box) |
| @gemini | 诊室高对比度模式 + 大点击区域 | ✅ 已纳入 (前端 UX 规范) |
| @gemini | DICOM 缩略图自动生成 | ✅ 已纳入 (thumbnail_url 字段) |
| @gemini | 电子签名拟物化印章标识 | ✅ 已纳入 (is_signed + signature_hash) |
| @gpt52 | 病历快照不可变模型 | ✅ 已纳入 (encounter_revisions) |
| @gpt52 | 事务边界一致性 | ✅ 已纳入 (revision_token) |
| @gpt52 | DICOM 三步上传闭环 | ✅ 已纳入 (upload_session -> finalize -> parse) |
| @gpt52 | AI 结果独立存储 | ✅ 已纳入 (ai_findings 表) |

### 9.3 设计原则总结

| 原则 | 实现 |
|------|------|
| **病历不可变性** | 签名后创建新 revision，禁止直接修改 |
| **事务边界一致** | encounters + 所有子表挂同一 revision_token |
| **DICOM 客户端非真相源** | 后端 upload_session -> finalize -> parse 闭环 |
| **AI 与诊断分离** | ai_findings 独立表，医生审核后关联 |
| **数据持久化策略** | PostgreSQL 是最终真相源，Redis 仅作缓存 |
| **多租户 UID 唯一性** | 租户内 study_uid/series_uid/sop_instance_uid 唯一约束 |

### 9.4 测试用例需求

| 场景 | 测试目标 |
|------|---------|
| 签名后编辑必须创建新 revision | 验证不可变性 |
| DICOM finalize 幂等 | 验证重复确认不重复处理 |
| 重复 SOP Instance UID 去重/拒绝 | 验证 UID 唯一约束 |
| 同一 encounter 并发编辑冲突检测 | 验证事务边界一致性 |

---

@gemini @gpt52
以上是 `emr-service` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: 牙位图交互、病历编辑器 UX、影像查看器动效是否需要补充？
- **@gpt52**: DICOM 解析流程是否完整？Outbox 模式是否有遗漏？数据一致性是否有风险？
