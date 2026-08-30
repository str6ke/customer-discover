# 外贸拓客系统 MVP 设计规格

## 1. 目标

构建一个可内测的导入型外贸拓客系统，将“产品与市场输入 → 关键词生成 → 线索导入 → 信息清洗 → 去重评分 → 客户表导出”打通，验证 AI 是否能显著减少外贸人员整理客户资料的重复劳动。

首期采用当前工作区已有的锅具产品资料作为真实演示数据，同时产品输入保持通用，未来可扩展到篷盖布、PVC、家具等品类。

## 2. 首期范围

### 2.1 包含

- 通用产品信息与目标市场输入
- 产品资料文件上传
- AI 关键词分组生成与人工编辑
- CSV、XLSX、TXT、Markdown、HTML 线索导入
- 公司名称、官网、域名、国家、城市、行业、联系人、职位、邮箱、电话、来源链接等字段抽取
- 公司名称、域名、邮箱标准化与多级去重
- 产品匹配度、市场匹配度、数据完整度、采购意图等维度评分
- A/B/C/D 客户等级
- 客户状态管理、人工修正和来源核验
- 按筛选条件导出 XLSX
- 任务进度、失败原因、重试和原始数据追溯

### 2.2 不包含

- Google、LinkedIn 或海关数据库的实时自动抓取
- 邮件、LinkedIn、WhatsApp 自动发送
- 自动跟进和行为触发
- GDPR 合法发送判断与发送频率控制
- 多租户计费、团队权限和复杂 CRM 同步

## 3. 总体架构

采用“模块化单体 + Agent 接口预留”。前端使用 Vue 3 与 Element Plus；后端使用 FastAPI；PostgreSQL 保存项目、线索、评分和任务数据；Redis 支持异步任务与缓存；LiteLLM 统一接入 OpenAI、DeepSeek、通义千问等模型。

```text
Vue3 + Element Plus
        │
        ▼
FastAPI API 层
        │
 ┌──────┼────────┐
 ▼      ▼        ▼
产品模块  线索模块  任务模块
        │
        ▼
AI 服务层（LiteLLM）
        │
 ┌──────┼──────────────┐
 ▼      ▼              ▼
关键词生成  字段抽取/清洗  客户评分
        │
        ▼
PostgreSQL
```

核心模块边界：

- `product`：产品资料与目标市场
- `keyword`：关键词生成与人工编辑
- `importer`：多格式线索导入
- `extractor`：字段识别、标准化与清洗
- `deduplication`：公司、域名和邮箱去重
- `scoring`：客户等级与评分解释
- `export`：客户表导出
- `jobs`：异步任务状态、错误和重试
- `ai`：LiteLLM 统一模型接口
- `agent_contracts`：后续多 Agent 的统一输入输出协议

## 4. 数据流

1. 用户填写产品名称、产品描述、核心参数、卖点、认证、环保属性、目标国家/地区和客户类型。
2. `KeywordAgent` 生成产品词、应用场景词、客户身份词、采购意图词和国家/语言变体词。
3. 用户上传网页搜索结果或现有客户表。
4. `ImportAgent` 保存原始内容，并将文件转为统一记录。
5. `AnalysisAgent` 提取和标准化客户字段。
6. `DedupAgent` 基于标准化公司名、域名、邮箱执行去重。
7. `ScoringAgent` 计算分项分数、总分、等级和解释。
8. 用户筛选、修正、标记客户状态并核验来源。
9. `ExportAgent` 生成 XLSX 文件。

每个阶段都保存任务状态和处理日志。原始导入内容永不覆盖；AI 结果必须可人工修改；重要字段保留来源链接。

## 5. 数据模型

### `projects`

保存一次获客项目：`id`、项目名称、产品类别、目标国家、客户类型、状态、创建时间。

### `products`

保存产品资料：`name`、描述、核心参数、卖点、认证、环保属性、产品文档路径。

### `keyword_sets`

保存关键词方案：`project_id`、关键词文本、关键词类型、目标语言、来源、用户是否确认。

### `import_jobs`

保存导入任务：`project_id`、文件名、文件类型、处理状态、成功数、失败数、错误日志。

### `leads`

保存客户线索：`company_name`、`normalized_company_name`、`website`、`domain`、`country`、`city`、`industry`、`contact_name`、`job_title`、`email`、`phone`、`source_url`、`source_type`、`raw_data`。

### `lead_evaluations`

保存客户评分：`lead_id`、`total_score`、`level`、`product_fit_score`、`market_fit_score`、`data_quality_score`、`buying_intent_score`、`reasoning`。

### `lead_status_history`

保存客户状态记录：`lead_id`、状态、备注、操作人、时间。

客户状态为：`新导入 → 待核验 → 已确认 → 已排除 → 已导出`。

客户等级为：

- A：高匹配、高价值或采购意图明显
- B：具备潜力，但需要人工进一步核验
- C：信息不足或匹配度较低
- D：重复、无效或明显不相关

## 6. Agent 契约

首期使用清晰的服务接口实现能力，后续再接入 LangGraph 编排。

```python
class AgentContext:
    project_id: str
    task_id: str
    input_data: dict
    metadata: dict

class AgentResult:
    status: str
    data: dict
    evidence: list[dict]
    warnings: list[str]
    errors: list[str]
```

首期 Agent：`KeywordAgent`、`ImportAgent`、`AnalysisAgent`、`DedupAgent`、`ScoringAgent`、`ExportAgent`。

后续 Agent：`SearchAgent`、`ContentAgent`、`ExecutionAgent`、`FollowupAgent`、`ComplianceAgent`、`OrchestratorAgent`。

## 7. 首期页面

1. 项目首页：项目卡片、处理进度、线索数量、等级分布和最近任务。
2. 新建获客项目：产品信息、目标市场、客户类型、资料上传、获客目标。
3. 关键词工作台：分组查看、编辑、删除、添加、语言筛选和确认。
4. 线索导入页：文件拖拽、字段预览、列映射和抽样检查。
5. 客户资料表：筛选、排序、批量修改、评分解释、来源链接和人工补充。
6. 客户详情抽屉：原始数据、清洗结果、评分明细、来源证据和操作记录。
7. 导出中心：按等级、国家、状态、字段完整度筛选并导出 XLSX。

AI 生成内容显示“AI 建议”标识；删除采用“标记排除”，不删除原始数据。

## 8. 异常处理

- 文件格式错误：任务失败并提示支持格式。
- 字段无法识别：保留原始内容，进入待人工映射状态。
- AI 调用失败：指数退避重试，超过次数后转人工处理。
- 单条线索解析失败：记录失败原因，不影响其他线索。
- 重复客户：保留主记录，其余记录进入重复记录表。
- 评分信息不足：降低数据质量分，不虚构采购信息。
- 导出失败：保留筛选条件，允许重新导出。

## 9. 测试与验收

自动化测试覆盖产品输入校验、关键词结构、五种导入格式、字段标准化、多级去重、评分计算、评分解释、XLSX 导出、任务失败和重试。

内测目标：

- 线索字段解析准确率 ≥ 90%
- 重复线索识别准确率 ≥ 95%
- A/B 客户人工认可率 ≥ 70%
- 单次 1,000 条线索处理成功率 ≥ 95%
- 从导入到导出的人工作业时间减少 ≥ 60%
- 关键任务失败后可恢复，且不丢失原始数据

## 10. 技术约束

- 后端：FastAPI、Pydantic、SQLAlchemy、Alembic
- 前端：Vue 3、TypeScript、Element Plus
- 数据库：PostgreSQL
- 异步任务：Redis + 独立 Worker
- AI：LiteLLM，使用结构化 JSON 输出
- 导出：XLSX 生成库
- 部署：Docker Compose

## 11. 后续演进

完成 MVP 内测后，按真实反馈依次扩展：搜索与行业数据连接器 → 客户画像与采购历史 → 个性化内容 → 单渠道邮件触达 → 跟进自动化 → 合规 Agent → 多渠道触达 → LangGraph 调度层。
