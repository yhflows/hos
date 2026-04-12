# clinic-service 详细设计草案

> 状态: 草案 - 待团队评审
> 作者: @opus (布偶猫)
> 创建日期: 2026-04-12

---

## 1. 服务定位

`clinic-service` 是 HOS 系统中基础数据管理的核心服务，负责诊所配置、医生管理、科室管理、患者档案、排班管理。本服务为 `booking-service`、`emr-service` 提供基础数据支撑。

### 1.1 职责边界

| 职责 | 负责 | 不负责 |
|------|------|--------|
| 诊所配置管理 | ✅ | - |
| 医生基础信息管理 | ✅ | - |
| 科室管理 | ✅ | - |
| 患者档案管理 | ✅ | - |
| 排班计划管理 | ✅ | - |
| 号源可用性查询 | ✅ | - |
| 医生状态管理 | ✅ | - |
| 预约号源扣减 | ❌ | booking-service |
| 预约状态流转 | ❌ | booking-service |
| 就诊记录管理 | ❌ | emr-service |
| 账单结算 | ❌ | payment-service |

### 1.2 核心约束

- **多租户隔离**: 所有操作必须在租户 schema 内执行
- **数据一致性**: 排班数据变更需考虑已预约的影响
- **审计日志**: 医生、科室、排班关键操作必须记录审计日志
- **状态单调性**: 医生状态、排班状态必须遵循状态机规则
- **幂等性**: 患者创建、排班创建需要幂等保证

---

## 2. 数据模型

### 2.1 租户隔离

采用 `Schema-per-tenant` 模式，每个租户有独立的 schema：`tenant_{tenant_id}`

### 2.2 核心表结构

#### clinics (诊所配置表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, UNIQUE, IDX |
| name | VARCHAR | 诊所名称 | - |
| code | VARCHAR | 诊所编码 | UNIQUE, IDX |
| address | TEXT | 地址 | - |
| phone | VARCHAR | 电话 | - |
| business_hours | JSONB | 营业时间 | - |
| status | VARCHAR | 状态 (active/inactive/closed) | IDX |
| config | JSONB | 配置项（预约规则、退费规则等） | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id)` - 租户查询
- `(tenant_id, code)` - 唯一约束
- `(tenant_id, status)` - 启用诊所查询

#### departments (科室表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| clinic_id | VARCHAR | 诊所 ID | FK, IDX |
| name | VARCHAR | 科室名称 | - |
| code | VARCHAR | 科室编码 | UNIQUE, IDX |
| category | VARCHAR | 科室分类 (clinical/technical/admin) | IDX |
| sort_order | INT | 排序 | - |
| status | VARCHAR | 状态 (active/inactive) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id, clinic_id, status)` - 诊所科室查询
- `(tenant_id, code)` - 唯一约束
- `(tenant_id, category, status)` - 分类查询

#### doctors (医生表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| clinic_id | VARCHAR | 诊所 ID | FK, IDX |
| department_id | VARCHAR | 科室 ID | FK, IDX |
| user_id | VARCHAR | 关联用户 ID | UNIQUE, IDX |
| code | VARCHAR | 医生编码 | UNIQUE, IDX |
| name | VARCHAR | 医生姓名 | - |
| title | VARCHAR | 职称 | - |
| specialty | VARCHAR | 专长 | - |
| phone | VARCHAR | 电话 | - |
| status | VARCHAR | 状态 (active/leave/resigned) | IDX |
| current_queue_position | INT | 当前排队位置 | - |
| current_encounter_id | VARCHAR | 当前就诊 ID | IDX |
| config | JSONB | 配置项（预约上限、加号规则等） | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id, clinic_id, status)` - 诊所医生查询
- `(tenant_id, department_id, status)` - 科室医生查询
- `(tenant_id, user_id)` - 用户关联
- `(tenant_id, code)` - 唯一约束

#### doctor_status_history (医生状态历史表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| doctor_id | VARCHAR | 医生 ID | FK, IDX |
| from_status | VARCHAR | 原状态 | - |
| to_status | VARCHAR | 新状态 | - |
| reason | VARCHAR | 变更原因 | - |
| operator_id | VARCHAR | 操作人 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**状态机**:
```
active → leave: 请假
active → resigned: 离职
leave → active: 复岗
```

#### patients (患者档案表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| clinic_id | VARCHAR | 默认诊所 ID | FK, IDX |
| user_id | VARCHAR | 关联用户 ID（可为空） | UNIQUE, IDX |
| patient_no | VARCHAR | 患者编号 | UNIQUE, IDX |
| name | VARCHAR | 姓名 | - |
| gender | VARCHAR | 性别 | - |
| birth_date | DATE | 出生日期 | - |
| id_card | VARCHAR | 身份证号 | - |
| phone | VARCHAR | 手机号 | IDX |
| email | VARCHAR | 邮箱 | - |
| address | TEXT | 地址 | - |
| emergency_contact | JSONB | 紧急联系人 | - |
| allergy_info | TEXT | 过敏信息 | - |
| medical_history | TEXT | 既往病史 | - |
| status | VARCHAR | 状态 (active/inactive/blacklist) | IDX |
| tags | JSONB | 标签 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id, phone)` - 手机号查询（常用）
- `(tenant_id, patient_no)` - 唯一约束
- `(tenant_id, id_card)` - 身份证查询
- `(tenant_id, name, phone)` - 姓名手机组合查询

**数据库硬约束**:
- `UNIQUE(tenant_id, patient_no)` - 患者编号唯一
- `UNIQUE(tenant_id, phone, status)` WHERE status = 'active' - 同一手机号不能有多个活跃患者

#### schedules (排班表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| clinic_id | VARCHAR | 诊所 ID | FK, IDX |
| doctor_id | VARCHAR | 医生 ID | FK, IDX |
| department_id | VARCHAR | 科室 ID | FK, IDX |
| work_date | DATE | 工作日期 | IDX |
| shift_type | VARCHAR | 班次类型 (morning/afternoon/evening/full_day) | IDX |
| shift_start_time | TIME | 班次开始时间 | - |
| shift_end_time | TIME | 班次结束时间 | - |
| capacity | INT | 总容量 | - |
| booked_count | INT | 已预约数 | - |
| available_count | INT | 可用号源 (capacity - booked_count) | - |
| is_repeatable | BOOLEAN | 是否重复排班 | - |
| repeat_pattern | JSONB | 重复模式（周几） | - |
| repeat_end_date | DATE | 重复结束日期 | - |
| status | VARCHAR | 状态 (active/cancelled/locked) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |
| updated_at | TIMESTAMP | 更新时间 | IDX |

**索引策略**:
- `(tenant_id, doctor_id, work_date, status)` - 医生排班查询
- `(tenant_id, department_id, work_date, status)` - 科室排班查询
- `(tenant_id, work_date, status)` - 日期排班查询

**状态机**:
```
active → cancelled: 取消排班（考虑已预约影响）
active → locked: 锁定排班（临时调整）
cancelled → active: 恢复排班
```

**数据库硬约束**:
- `CHECK(booked_count >= 0 AND booked_count <= capacity)` - 预约数校验
- `CHECK(available_count >= 0)` - 可用号源校验

#### schedule_time_slots (排班时段表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| schedule_id | VARCHAR | 排班 ID | FK, IDX |
| time_slot | TIME | 时段（如 09:00, 09:30） | IDX |
| capacity | INT | 时段容量 | - |
| booked_count | INT | 已预约数 | - |
| available_count | INT | 可用号源 | - |
| status | VARCHAR | 状态 (available/locked/cancelled) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, schedule_id, time_slot, status)` - 时段查询

**数据库硬约束**:
- `CHECK(booked_count >= 0 AND booked_count <= capacity)` - 预约数校验

#### schedule_reservations (排班预占表，用于 booking-service 预占号源)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| schedule_id | VARCHAR | 排班 ID | FK, IDX |
| time_slot_id | VARCHAR | 时段 ID | FK, IDX |
| reservation_key | VARCHAR | 预占键（来自 booking-service） | UNIQUE, IDX |
| reserved_at | TIMESTAMP | 预占时间 | IDX |
| expires_at | TIMESTAMP | 过期时间 | IDX |
| status | VARCHAR | 状态 (pending/confirmed/failed/expired) | IDX |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, schedule_id, time_slot_id, status)` - 段预占查询
- `(tenant_id, reservation_key)` - 唯一约束
- `(tenant_id, expires_at, status)` - 过期清理查询

#### audit_logs (审计日志表)

| 字段名 | 类型 | 说明 | 索引 |
|--------|------|------|------|
| id | ULID | 主键 | PK |
| tenant_id | VARCHAR | 租户 ID | FK, IDX |
| user_id | VARCHAR | 操作人 ID | IDX |
| action | VARCHAR | 操作类型 | IDX |
| resource_type | VARCHAR | 资源类型 (doctor/patient/schedule) | IDX |
| resource_id | VARCHAR | 资源 ID | - |
| old_value | JSONB | 旧值 | - |
| new_value | JSONB | 新值 | - |
| ip_address | VARCHAR | IP 地址 | - |
| created_at | TIMESTAMP | 创建时间 | IDX |

**索引策略**:
- `(tenant_id, user_id, created_at DESC)` - 用户操作历史
- `(tenant_id, resource_type, resource_id, created_at DESC)` - 资源变更历史

---

## 3. API 接口设计

### 3.1 诊所管理

#### GET /api/v1/clinics

获取诊所列表

**查询参数**:
- `status`: 状态过滤
- `page`: 页码
- `page_size`: 每页数量

**响应**:
```json
{
  "total": 10,
  "items": [
    {
      "id": "clinic_001",
      "name": "阳光口腔诊所",
      "code": "CL001",
      "address": "北京市朝阳区...",
      "phone": "010-12345678",
      "status": "active"
    }
  ]
}
```

#### GET /api/v1/clinics/:id

获取诊所详情

### 3.2 科室管理

#### GET /api/v1/departments

获取科室列表

**查询参数**:
- `clinic_id`: 诊所 ID
- `category`: 分类过滤
- `status`: 状态过滤

**响应**:
```json
{
  "total": 5,
  "items": [
    {
      "id": "dept_001",
      "name": "种植科",
      "code": "D001",
      "category": "clinical",
      "status": "active"
    }
  ]
}
```

#### POST /api/v1/departments

创建科室

**请求体**:
```json
{
  "clinic_id": "clinic_001",
  "name": "种植科",
  "code": "D001",
  "category": "clinical"
}
```

### 3.3 医生管理

#### GET /api/v1/doctors

获取医生列表

**查询参数**:
- `clinic_id`: 诊所 ID
- `department_id`: 科室 ID
- `status`: 状态过滤
- `keyword`: 姓名/编码搜索

**响应**:
```json
{
  "total": 20,
  "items": [
    {
      "id": "doctor_001",
      "name": "张医生",
      "code": "D001",
      "title": "主任医师",
      "specialty": "口腔种植",
      "status": "active",
      "current_queue_position": 3,
      "current_encounter_id": null
    }
  ]
}
```

#### GET /api/v1/doctors/:id

获取医生详情

**响应**:
```json
{
  "id": "doctor_001",
  "name": "张医生",
  "title": "主任医师",
  "specialty": "口腔种植",
  "phone": "13800138000",
  "status": "active",
  "clinic": {
    "id": "clinic_001",
    "name": "阳光口腔诊所"
  },
  "department": {
    "id": "dept_001",
    "name": "种植科"
  },
  "config": {
    "max_daily_appointments": 30,
    "allow_overbooking": false
  }
}
```

#### POST /api/v1/doctors/:id/status

修改医生状态

**请求体**:
```json
{
  "status": "leave",
  "reason": "休假"
}
```

### 3.4 患者管理

#### GET /api/v1/patients

获取患者列表

**查询参数**:
- `clinic_id`: 诊所 ID
- `keyword`: 姓名/手机号搜索
- `status`: 状态过滤
- `page`: 页码
- `page_size`: 每页数量

**响应**:
```json
{
  "total": 100,
  "items": [
    {
      "id": "patient_001",
      "patient_no": "P001",
      "name": "李四",
      "gender": "male",
      "birth_date": "1990-01-01",
      "phone": "13800138000",
      "status": "active",
      "tags": ["VIP", "慢性病患者"]
    }
  ]
}
```

#### GET /api/v1/patients/:id

获取患者详情

**响应**:
```json
{
  "id": "patient_001",
  "patient_no": "P001",
  "name": "李四",
  "gender": "male",
  "birth_date": "1990-01-01",
  "id_card": "110101199001011234",
  "phone": "13800138000",
  "email": "lisi@example.com",
  "address": "北京市朝阳区...",
  "emergency_contact": {
    "name": "王五",
    "phone": "13900139000",
    "relation": "配偶"
  },
  "allergy_info": "青霉素过敏",
  "medical_history": "无",
  "status": "active",
  "tags": ["VIP"]
}
```

#### POST /api/v1/patients

创建患者

**请求体**:
```json
{
  "name": "李四",
  "gender": "male",
  "birth_date": "1990-01-01",
  "phone": "13800138000",
  "id_card": "110101199001011234",
  "email": "lisi@example.com",
  "address": "北京市朝阳区..."
}
```

#### PUT /api/v1/patients/:id

更新患者信息

### 3.5 排班管理

#### GET /api/v1/schedules

查询排班

**查询参数**:
- `doctor_id`: 医生 ID
- `department_id`: 科室 ID
- `clinic_id`: 诊所 ID
- `date_from`: 起始日期
- `date_to`: 结束日期
- `status`: 状态过滤

**响应**:
```json
{
  "total": 30,
  "items": [
    {
      "id": "schedule_001",
      "doctor": {
        "id": "doctor_001",
        "name": "张医生"
      },
      "department": {
        "id": "dept_001",
        "name": "种植科"
      },
      "work_date": "2026-04-15",
      "shift_type": "morning",
      "shift_start_time": "09:00",
      "shift_end_time": "12:00",
      "capacity": 10,
      "booked_count": 5,
      "available_count": 5,
      "status": "active",
      "time_slots": [
        {
          "id": "slot_001",
          "time_slot": "09:00",
          "capacity": 2,
          "booked_count": 1,
          "available_count": 1,
          "status": "available"
        }
      ]
    }
  ]
}
```

#### GET /api/v1/schedules/available

查询可用号源

**查询参数**:
- `doctor_id`: 医生 ID
- `department_id`: 科室 ID
- `date_from`: 起始日期
- `date_to`: 结束日期

**响应**:
```json
{
  "available": [
    {
      "schedule_id": "schedule_001",
      "date": "2026-04-15",
      "shift": "上午",
      "available_slots": [
        {
          "slot_id": "slot_001",
          "time": "09:00",
          "available": 1
        },
        {
          "slot_id": "slot_002",
          "time": "09:30",
          "available": 2
        }
      ]
    }
  ]
}
```

#### POST /api/v1/schedules

创建排班

**请求体**:
```json
{
  "clinic_id": "clinic_001",
  "doctor_id": "doctor_001",
  "department_id": "dept_001",
  "work_date": "2026-04-15",
  "shift_type": "morning",
  "shift_start_time": "09:00",
  "shift_end_time": "12:00",
  "capacity": 10,
  "time_slots": [
    {"time_slot": "09:00", "capacity": 2},
    {"time_slot": "09:30", "capacity": 2},
    {"time_slot": "10:00", "capacity": 2}
  ],
  "is_repeatable": false
}
```

#### POST /api/v1/schedules/batch

批量创建排班

**请求体**:
```json
{
  "clinic_id": "clinic_001",
  "doctor_id": "doctor_001",
  "department_id": "dept_001",
  "date_range": {
    "from": "2026-04-15",
    "to": "2026-04-30"
  },
  "repeat_pattern": ["monday", "wednesday", "friday"],
  "shift_type": "morning",
  "shift_start_time": "09:00",
  "shift_end_time": "12:00",
  "capacity": 10,
  "time_slots": [
    {"time_slot": "09:00", "capacity": 2},
    {"time_slot": "09:30", "capacity": 2}
  ]
}
```

#### POST /api/v1/schedules/:id/reserve

预占号源（供 booking-service 调用）

**请求体**:
```json
{
  "time_slot_id": "slot_001",
  "reservation_key": "res_abc123",
  "expires_at": "2026-04-12T10:00:00Z"
}
```

**响应**:
```json
{
  "reserved": true,
  "reservation_key": "res_abc123",
  "expires_at": "2026-04-12T10:00:00Z"
}
```

#### POST /api/v1/schedules/:id/confirm

确认预占（供 booking-service 调用）

**请求体**:
```json
{
  "reservation_key": "res_abc123",
  "appointment_id": "apt_abc123"
}
```

#### PUT /api/v1/schedules/:id/cancel

取消排班

**请求体**:
```json
{
  "reason": "医生临时请假"
}
```

---

## 4. gRPC 服务定义

### 4.1 Clinic Service

```protobuf
service ClinicService {
  // 诊所查询
  rpc GetClinic(GetClinicRequest) returns (Clinic);
  rpc ListClinics(ListClinicsRequest) returns (ListClinicsResponse);

  // 科室查询
  rpc GetDepartment(GetDepartmentRequest) returns (Department);
  rpc ListDepartments(ListDepartmentsRequest) returns (ListDepartmentsResponse);

  // 医生查询
  rpc GetDoctor(GetDoctorRequest) returns (Doctor);
  rpc ListDoctors(ListDoctorsRequest) returns (ListDoctorsResponse);
  rpc GetDoctorStatus(GetDoctorStatusRequest) returns (DoctorStatus);

  // 患者查询
  rpc GetPatient(GetPatientRequest) returns (Patient);
  rpc ListPatients(ListPatientsRequest) returns (ListPatientsResponse);
  rpc FindPatientByPhone(FindPatientByPhoneRequest) returns (Patient);

  // 排班查询
  rpc GetSchedule(GetScheduleRequest) returns (Schedule);
  rpc ListSchedules(ListSchedulesRequest) returns (ListSchedulesResponse);
  rpc GetAvailableSlots(GetAvailableSlotsRequest) returns (AvailableSlots);

  // 排班操作（供 booking-service 使用）
  rpc ReserveSlot(ReserveSlotRequest) returns (ReserveSlotResponse);
  rpc ConfirmReservation(ConfirmReservationRequest) returns (ConfirmReservationResponse);
  rpc CancelReservation(CancelReservationRequest) returns (google.protobuf.Empty);
}
```

---

## 5. Outbox 事件发布

### 5.1 事件类型

| 事件类型 | 触发时机 | 消费方 | 说明 |
|---------|---------|--------|------|
| `doctor.status_changed` | 医生状态变更 | push-service | 通知相关人员 |
| `schedule.created` | 排班创建 | push-service | 通知医生排班更新 |
| `schedule.cancelled` | 排班取消 | booking-service, push-service | 取消已预约、通知患者 |
| `schedule.capacity_changed` | 排班容量变更 | booking-service | 调整可用号源 |
| `patient.created` | 患者创建 | push-service | 欢迎新患者 |

### 5.2 事件数据结构

#### doctor.status_changed
```json
{
  "event_id": "evt_abc123",
  "tenant_id": "tenant_999",
  "doctor_id": "doctor_001",
  "from_status": "active",
  "to_status": "leave",
  "reason": "休假",
  "occurred_at": "2026-04-12T10:00:00Z"
}
```

#### schedule.cancelled
```json
{
  "event_id": "evt_abc123",
  "tenant_id": "tenant_999",
  "schedule_id": "schedule_001",
  "work_date": "2026-04-15",
  "doctor_id": "doctor_001",
  "reason": "医生临时请假",
  "affected_appointments": ["apt_001", "apt_002"],
  "occurred_at": "2026-04-12T10:00:00Z"
}
```

---

## 6. 依赖服务

| 服务 | 依赖方式 | 用途 |
|------|---------|------|
| `booking-service` | gRPC | 预占号源、确认预占、取消预约时扣减号源 |
| `push-service` | NATS | 发布医生状态、排班变更通知 |
| `PostgreSQL` | 直接连接 | 数据持久化 |
| `NATS JetStream` | 直接连接 | 发布 Outbox 事件 |

---

## 7. 安全与合规

### 7.1 权限控制

| 操作 | 所需角色 | 权限 |
|------|---------|------|
| 创建/修改医生 | admin | `clinic:manage_doctor` |
| 修改医生状态 | admin | `clinic:change_doctor_status` |
| 创建/修改科室 | admin | `clinic:manage_department` |
| 创建患者 | admin, patient | `clinic:create_patient` |
| 查看患者信息 | doctor, admin | `clinic:read_patient` |
| 创建/修改排班 | admin | `clinic:manage_schedule` |
| 查看排班 | patient, doctor, admin | `clinic:read_schedule` |
| 预占号源 | booking-service | 内部调用 |

### 7.2 安全措施

- **患者隐私保护**: 敏感信息（身份证、手机号）脱敏显示
- **数据加密**: 数据库敏感字段加密存储
- **审计日志**: 所有关键操作记录审计日志
- **防爬虫**: 接口限流，防止恶意爬取患者信息

---

## 8. 性能指标

| 指标 | 目标值 |
|------|--------|
| 医生列表查询响应时间 | < 100ms |
| 排班查询响应时间 | < 200ms |
| 可用号源查询响应时间 | < 300ms |
| 预占号源响应时间 | < 50ms |
| 确认预占响应时间 | < 50ms |
| 患者信息查询响应时间 | < 100ms |

---

## 9. 待讨论事项

1. **排班重复模式**: 是否支持更复杂的重复模式（如每月特定日期）？
2. **医生跨科室**: 医生是否可以跨科室排班？
3. **患者合并**: 同一患者在不同诊所如何合并档案？
4. **临时加号**: 医生是否可以临时加号（超过容量）？
5. **排班取消规则**: 已有预约的排班取消时如何处理？（强制取消、人工确认？）
6. **患者标签系统**: 标签是否需要预定义还是自由添加？

---

@gemini @gpt52
以上是 `clinic-service` 的详细设计草案。请从各自专业角度评审：
- **@gemini**: 患者管理 UX 是否完善？医生状态提示是否清晰？排班界面交互是否需要补充？
- **@gpt52**: 数据模型是否完整？索引策略是否合理？号源预占逻辑是否安全？是否有安全风险？
