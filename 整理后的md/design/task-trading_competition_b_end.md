# 交易大赛 - B端功能开发任务拆解

本文档基于 `新需求/整理后的md/new-交易大赛-B端.md` 及其对应的设计文档 `design/trading_competition_b_end_design.md`，旨在将B端（项目方）交易大赛管理功能分解为可执行的开发任务。任务拆解以功能模块为主要粒度，目标是实现满足核心需求的最小化可行产品（MVP）。

## 1. 核心数据模型与基础架构 (task-tb-001)

- **父任务**: 无
- **描述**:
  - 根据 `design/trading_competition_b_end_design.md` 中的数据库设计，创建以下核心表结构：
    - `trading_competitions`: 交易大赛活动主表。
    - `trading_competition_page_styles`: 页面样式配置表。
    - `trading_competition_projects`: 参与项目信息表。
    - `trading_competition_pools`: 奖池配置表。
    - `trading_competition_budget_deposits`: 预算充值记录表。
  - 搭建项目基础结构，包括相关的DTO（如 `TradingCompetitionBasicInfoParamsDTO`, `TradingCompetitionProjectParamsDTO`, `TradingCompetitionPoolParamsDTO`, `TokenIdentifierDTO`, `TradingCompetitionPageStyleParamsDTO` 等）和服务层骨架。
- **验收标准**:
  - 数据库表结构成功创建并通过迁移脚本管理。
  - 核心DTO定义完成。
  - 基础服务层接口和实现骨架搭建完成，可供后续模块注入和使用。
- **预计工作量**: 中

## 2. 活动创建与基本信息配置 (task-tb-002)

- **父任务**: task-tb-001
- **描述**: 实现B端创建新交易大赛活动的功能，并配置其基本信息。
  - 实现API： `tradingCompetitionAdmin_createTradingCompetition` (部分功能，仅处理活动基础信息、项目、奖池的骨架保存)。
  - 实现API： `tradingCompetitionAdmin_updateTradingCompetition` (部分功能，用于更新活动基本信息)。
  - 后端逻辑处理：保存活动名称、时间、Banner图URL、点状动画开关、全局网络和DEX设置。
  - 前端界面：提供表单用于输入活动名称、选择开始/结束日期、上传PC/Mobile Banner图、设置点状动画开关、选择全局交易网络和DEX。
- **涉及DTOs**: `CreateTradingCompetitionParamsDTO` (部分), `TradingCompetitionBasicInfoParamsDTO`, `TradingCompetitionDetailAdminDTO` (返回)。
- **数据库交互**: `trading_competitions` 表的创建和更新。
- **验收标准**:
  - B端用户可以通过API或管理后台界面成功创建一个新的交易大赛活动框架，并保存其基本信息。
  - 活动基本信息能正确存储到 `trading_competitions` 表中。
  - 可以更新已创建活动的上述基本信息。
- **预计工作量**: 中

## 3. 参与项目配置 (task-tb-003)

- **父任务**: task-tb-002
- **描述**: 实现B端在创建或编辑活动时配置参与项目信息的功能。
  - 后端逻辑：允许添加、编辑、删除、排序参与项目。保存项目名称、Logo URL、Twitter Handle、官网URL。
  - 支持从Twitter Handle自动抓取项目名称和Logo (调用 `tradingCompetitionAdmin_fetchProjectInfoByTwitter` API)。
  - 检查项目是否已发币，并引导创建奖池 (调用 `tradingCompetitionAdmin_checkProjectTokenLaunchStatus` API)。
  - 前端界面：提供列表管理参与项目，支持拖拽排序、编辑、删除。添加新项目时，提供通过Twitter Handle抓取或手动上传两种方式。
- **涉及DTOs**: `TradingCompetitionProjectParamsDTO`, `FetchProjectInfoByTwitterParamsDTO`, `FetchedProjectInfoDTO`, `CheckProjectTokenParamsDTO`, `ProjectTokenCheckResultDTO`.
- **数据库交互**: `trading_competition_projects` 表的增删改查。
- **验收标准**:
  - B端用户可以为活动配置一个或多个参与项目。
  - 项目信息（名称、Logo、Twitter等）能正确保存和展示。
  - 支持通过Twitter Handle获取项目信息并填充。
  - 能够提示已发币项目并引导配置奖池。
- **预计工作量**: 中

## 4. 奖池配置 (task-tb-004)

- **父任务**: task-tb-002
- **描述**: 实现B端配置交易大赛奖池的复杂规则。
  - 后端逻辑：
    - 配置全局交易网络和DEX (已在task-tb-002中部分涉及，此处确保关联到奖池)。
    - 支持创建和管理多个奖池 (Basic Pool 和 指定 Reward Pool)。
    - Basic Pool: 配置奖励Token、数量、Swap From/To (Any Token/指定Token，支持多选)、排除同底层资产交易开关、获奖资格 (交易量门槛/排行榜名次)。
    - 指定 Reward Pool: 配置奖励Token、数量、Swap From (Any Token)、Swap To (同奖励Token)、获奖资格。支持拖动排序、删除。
    - 处理单项目方与多项目方时Basic Pool的强制性/可选性逻辑。
  - 前端界面：提供清晰的界面配置各个奖池的参数，包括Token选择器、数量输入、交易对配置、获奖资格设置。支持动态添加/删除/排序指定Reward Pool。
- **涉及DTOs**: `TradingCompetitionPoolParamsDTO`, `TokenIdentifierDTO`, `TradingCompetitionPoolWinnerEligibilityParamsDTO`.
- **数据库交互**: `trading_competition_pools` 表的增删改查。
- **验收标准**:
  - B端用户可以为活动配置至少一个奖池，包括Basic Pool和可选的指定Reward Pool。
  - 奖池的各项参数（奖励、交易对、获奖资格等）能被正确保存和加载。
  - 根据项目方数量正确处理Basic Pool的逻辑。
- **预计工作量**: 大

## 5. 页面样式配置 (task-tb-005)

- **父任务**: task-tb-002
- **描述**: 实现B端配置C端活动页面的背景和主题色。
  - 后端逻辑：保存背景类型（颜色/图片）、背景值、主题色。
  - 前端界面：提供颜色选择器、图片上传控件来配置页面样式。
- **涉及DTOs**: `TradingCompetitionPageStyleParamsDTO`.
- **数据库交互**: `trading_competition_page_styles` 表的创建和更新。
- **验收标准**:
  - B端用户可以为活动配置页面背景（纯色或图片）和主题色。
  - 配置信息能正确保存到 `trading_competition_page_styles` 表。
- **预计工作量**: 小

## 6. 预算与充值管理 (task-tb-006)

- **父任务**: task-tb-004
- **描述**: 实现活动预算的计算、展示及Token充值记录功能。
  - 后端逻辑：
    - 根据配置的奖池信息（奖励Token和数量）估算总预算需求 (USD)。
    - 实现API：`tradingCompetitionAdmin_getCompetitionBudgetStatus`，返回当前预算状态。
    - 实现API：`tradingCompetitionAdmin_recordBudgetDeposit`，记录B端用户的充值信息 (代币、数量、TxHash等)，并更新已充值金额。 后端需要对TxHash进行基础校验（例如格式、是否已存在）。
  - 前端界面：
    - 在奖池配置完成后显示预估总预算。
    - 当预算不足时，显示 "Top up" 提示。
    - 提供界面供B端提交充值记录 (选择代币、输入数量、填写TxHash)。
    - 展示当前已充值总额和还需充值金额。
- **涉及DTOs**: `CompetitionBudgetStatusDTO`, `DepositTokenParamsDTO`, `DepositedTokenInfoDTO`.
- **数据库交互**: `trading_competition_budget_deposits` 表的创建；`trading_competitions` 表的 `total_funded_usd` 更新。
- **验收标准**:
  - 系统能根据奖池配置估算总预算。
  - B端用户可以记录充值信息，系统能更新已充值总额。
  - 能正确显示预算是否充足。
- **预计工作量**: 中

## 7. 活动列表、状态管理与发布 (task-tb-007)

- **父任务**: task-tb-001
- **描述**: 实现B端查看和管理活动列表，以及活动状态变更（保存草稿、发布）。
  - 实现API： `tradingCompetitionAdmin_listTradingCompetitions`，支持分页和按状态/名称筛选。
  - 实现API： `tradingCompetitionAdmin_getTradingCompetitionDetailAdmin`，获取活动完整详情。
  - 实现API： `tradingCompetitionAdmin_deleteTradingCompetition` (仅限DRAFT, UPCOMING状态)。
  - 实现API： `tradingCompetitionAdmin_publishTradingCompetition`。
    - 发布前校验：预算金额是否已充值完毕；是否设置了至少一个奖池。校验失败则返回错误信息。
    - 发布成功后，活动状态根据开始时间变为 `UPCOMING` 或 `ONGOING`。
  - 后端逻辑：支持活动状态流转 (DRAFT, UPCOMING, ONGOING, ENDED等)。不同状态下活动的可编辑性不同。
  - 前端界面：
    - 展示活动列表，包含活动名称、时间、状态、总预算、操作按钮 (编辑、删除、数据分析、发布)。
    - 编辑页面提供 "Save as draft" (不校验) 和 "Publish" (校验) 按钮。
    - 对于已发布的活动，根据状态限制编辑权限。
- **涉及DTOs**: `TradingCompetitionListItemDTO`, `TradingCompetitionDetailAdminDTO`.
- **数据库交互**: `trading_competitions` 表的状态更新、查询、删除。
- **验收标准**:
  - B端用户可以查看自己创建的活动列表，并进行筛选。
  - 可以获取活动详情。
  - 可以删除处于草稿或即将开始状态的活动。
  - 可以发布活动，发布前会进行预算和奖池校验。
  - 活动状态能正确流转。
- **预计工作量**: 大

## 8. B端数据分析 - 基础数据表和API骨架 (task-tb-008)

- **父任务**: task-tb-001
- **描述**: 搭建B端数据分析所需的基础数据表和API骨架。实际的数据填充和复杂计算可能依赖数据管道。
  - 根据 `design/trading_competition_b_end_design.md` 创建以下数据分析相关表：
    - `trading_competition_user_volume`: 用户在各奖池的交易量统计。
    - `trading_competition_daily_summary`: 每日汇总数据。
    - `trading_competition_raw_transactions`: 原始交易明细。
  - 实现API骨架 (可返回mock数据或空数据)：
    - `tradingCompetitionAdmin_getCompetitionDataSummary`
    - `tradingCompetitionAdmin_getCompetitionLeaderboard`
    - `tradingCompetitionAdmin_getCompetitionRawTransactions`
  - 定义相关DTOs: `CompetitionDataRequestParamsDTO`, `CompetitionSummaryMetricsDTO`, `TimeSeriesDataPointDTO`, `CompetitionDataSummaryDTO`, `PoolInfoTabDTO`, `LeaderboardEntryAdminDTO`, `RawTransactionListItemDTO`.
- **数据库交互**: 创建上述三个数据分析表。API暂时不直接写入，依赖后续数据管道。
- **验收标准**:
  - 数据分析相关的数据库表已创建。
  - 数据分析模块的API接口已定义并可调用（可返回空数据或mock数据）。
  - 相关DTO已定义。
- **预计工作量**: 中

## 9. B端数据分析 - 汇总数据展示 (task-tb-009)

- **父任务**: task-tb-008
- **描述**: 实现B端数据分析模块中"汇总数据"的展示。
  - 后端逻辑： `tradingCompetitionAdmin_getCompetitionDataSummary` API 能够从 `trading_competition_daily_summary` 表中读取数据，并按需聚合。
  - 前端界面：
    - 提供Tab切换总览及各Pool数据。
    - 展示关键指标：Volume, Participants, Transactions 及日增数据。
    - 展示日趋势图 (Volume, Participants, Transactions)，支持7D/30D/All范围筛选，双Y轴。
- **数据来源**: `trading_competition_daily_summary` (假设由数据管道填充)。
- **验收标准**:
  - B端用户可以在数据分析页面查看活动的汇总数据和趋势图。
  - 支持按Pool切换查看。
  - 数据展示准确，符合需求文档描述。
- **预计工作量**: 中 (依赖数据管道)

## 10. B端数据分析 - Leaderboard数据展示 (task-tb-010)

- **父任务**: task-tb-008
- **描述**: 实现B端数据分析模块中"Leaderboard数据"的展示。
  - 后端逻辑： `tradingCompetitionAdmin_getCompetitionLeaderboard` API 能够从 `trading_competition_user_volume` 表中读取数据，并按交易量排序。支持分页。
  - 前端界面：
    - 提供Tab切换各Pool数据。
    - 表格展示排名、用户地址、Trading Volume、Transactions。
    - 支持导出数据 (MVP阶段可简化为前端导出当前页)。
    - 支持分页。
- **数据来源**: `trading_competition_user_volume` (假设由数据管道填充)。
- **验收标准**:
  - B端用户可以查看各奖池的Leaderboard。
  - 数据展示准确，包含排名、用户、交易量、交易笔数。
  - 支持分页和导出。
- **预计工作量**: 中 (依赖数据管道)

## 11. B端数据分析 - 明细交易数据展示 (task-tb-011)

- **父任务**: task-tb-008
- **描述**: 实现B端数据分析模块中"明细交易数据"的展示。
  - 后端逻辑：`tradingCompetitionAdmin_getCompetitionRawTransactions` API 能够从 `trading_competition_raw_transactions` 表中读取数据。支持分页。
  - 前端界面：
    - 提供Tab切换总览及各Pool数据。
    - 表格展示Wallet, Chain, From, To, Time, Hash。
    - 支持导出数据 (MVP阶段可简化为前端导出当前页)。
- **数据来源**: `trading_competition_raw_transactions` (假设由数据管道填充)。
- **验收标准**:
  - B端用户可以查看活动的明细交易数据。
  - 数据展示准确，包含所需字段。
  - 支持分页和导出。
- **预计工作量**: 中 (依赖数据管道)

## 12. 业务规则实现与校验 (task-tb-012)

- **父任务**: task-tb-002, task-tb-003, task-tb-004, task-tb-007
- **描述**: 在各个功能模块的后端逻辑中，实现需求文档中定义的业务规则。
  - 单项目方Basic Pool强制性 (BR-TB-001)。
  - 多项目方Basic Pool可选性 (BR-TB-002)。
  - 指定Reward Pool添加条件 (BR-TB-003)。
  - 项目发币与奖池引导 (BR-TB-004) - 提示逻辑。
  - Basic Pool交易对配置规则 (BR-TB-005, BR-TB-006)。
  - 指定Reward Pool交易对固定 (BR-TB-007)。
  - 获奖资格设置规则 (BR-TB-008)。
  - 预算校验与充值提示 (BR-TB-009) - 提示逻辑。
  - 发布活动校验 (BR-TB-010) - 已在task-tb-007中实现。
  - 活动状态与可编辑性 (BR-TB-011) - 在各更新API中校验。
  - 数据分析Pool顺序 (BR-TB-012) - 在数据分析API获取数据时注意排序。
- **验收标准**:
  - 所有列出的业务规则在相关操作中得到正确执行和校验。
  - 不符合规则的操作被阻止，并给出合适的提示。
- **预计工作量**: 中 (分散在多个任务中)

**后续任务 (MVP后考虑):**

- 所见即所得编辑器和实时预览功能。
- 更完善的Token充值流程和链上校验。
- 数据分析模块的导出功能（后端实现）。
- 数据管道的实际搭建和数据填充。
- 更细致的权限管理。
- 与C端活动页面的数据同步和展示。

---

**说明:**

- 每个任务都应包含相应的单元测试和集成测试。
- 前端任务与后端API开发紧密相关，建议并行或紧随其后进行。
- "预计工作量"是相对估算，具体会受团队熟悉度和技术复杂度影响。
- 数据分析部分（task-tb-009, task-tb-010, task-tb-011）强依赖数据管道对相应表格 (`trading_competition_user_volume`, `trading_competition_daily_summary`, `trading_competition_raw_transactions`) 的数据填充，在数据管道未就绪前，这些API可能只能返回mock数据或空数据。
