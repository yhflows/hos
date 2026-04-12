# ADR-002: 医疗影像标准 (Medical Imaging Standard)

## 1. 状态 (Status)
已接受 (Accepted) - 2026-04-12

## 2. 上下文 (Context)
项目需要存储和展示大量的牙片、CT 等医疗影像文件。原本的设计仅将其视为普通附件。

## 3. 决策结果 (Decision)
**方案: 医疗级 DICOM 集成模型**

*   **标准**: 采用 DICOM P10 格式作为核心标准。
*   **前端展示**: 引入 **CornerstoneJS** 或 **OHIF Viewer**。支持 Web 端查看、多窗口对比、窗宽窗位调节。
*   **数据索引**: 在数据库中建立 `imaging_studies` 表，参考 FHIR 模型存储 StudyInstanceUID, SeriesInstanceUID 等元数据。
*   **存储路径**: 前端上传 DICOM -> 后端解析解析并存入 MinIO -> 数据库记录元数据。
*   **影像服务 (Phase 2)**: 引入 **Orthanc** 服务器作为轻量级 PACS 节点，支持 DICOM Query/Retrieve (Q/R)。

## 4. 后果 (Consequences)
*   增加了前端 Bundle 大小（需要引入解析库）。
*   后端需要额外的 Go 库 (如 `suyashkumar/dicom`) 进行数据解析。
*   为未来的第三方影像系统 (PACS) 接入打下坚实基础。
