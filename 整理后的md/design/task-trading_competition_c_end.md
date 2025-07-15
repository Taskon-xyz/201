# 交易大赛 - C端功能开发任务拆解

本文档基于 `新需求/整理后的md/new-交易大赛-C端.md` 及其对应的设计文档 `design/trading_competition_c_end_design.md`，旨在将C端（用户参与方）交易大赛功能分解为可执行的开发任务。任务拆解以功能模块为主要粒度，目标是实现满足核心需求的最小化可行产品（MVP）。

## 1. 核心数据模型与基础架构 (task-tc-001)

-   **父任务**: 无 (或依赖B端核心表 task-tb-001)
-   **描述**:
    -   根据 `design/trading_competition_c_end_design.md` 中的数据库设计，创建C端特有的核心表：
        -   `trading_competition_user_rewards`: 交易大赛用户奖励表 (记录用户获奖和领取情况)。
    -   搭建项目基础结构，包括相关的C端DTO (如 `CompetitionDetailForUserDTO`, `UserCompetitionDataDTO`, `LeaderboardEntryDTO`, `UserRewardItemDTO`, `SwapContextDTO` 等) 和服务层骨架。
    -   C端大部分数据将读取自B端已创建的表，如 `trading_competitions`, `trading_competition_projects`, `trading_competition_pools`。数据分析结果可能读取自 `trading_competition_user_volume` (由数据管道填充)。
-   **验收标准**:
    -   `trading_competition_user_rewards` 表结构成功创建并通过迁移脚本管理。
    -   核心C端DTO定义完成。
    -   基础服务层接口和实现骨架搭建完成。
-   **预计工作量**: 中

## 2. 活动详情与规则展示 (task-tc-002)

-   **父任务**: task-tc-001
-   **描述**: 实现C端用户查看交易大赛的详细信息和规则。
    -   API: `tradingCompetitionUser_getCompetitionDetail` - 获取活动详情 (Banner, 名称, 时间状态, 主办方, 页面样式, 网络/DEX, 项目, 规则概述, 奖池规则, 总奖金, 总交易量, 总参与人数)。
    -   前端界面: 
        -   页面头部展示活动Banner、名称、时间状态、主办方、分享按钮、规则按钮。
        -   点击规则按钮弹出活动规则弹窗，展示总规则和各奖池规则。
        -   页面背景和主题色根据B端配置应用。
-   **涉及DTOs**: `CompetitionDetailForUserDTO`, `CompetitionProjectInfoDTO`, `CompetitionPoolRuleDTO`, `TradingCompetitionPageStyleParamsDTO`.
-   **数据库交互**: 读取 `trading_competitions`, `trading_competition_projects`, `trading_competition_pools`, `trading_competition_page_styles` 表。
-   **验收标准**:
    -   用户可以查看到活动的完整基本信息、项目信息、各奖池规则。
    -   页面样式能正确应用。
    -   规则弹窗能正确展示。
-   **预计工作量**: 中

## 3. Swap交互区域与钱包集成 (task-tc-003)

-   **父任务**: task-tc-001
-   **描述**: 实现用户连接钱包并在活动页面进行符合规则的Swap交易。
    -   API: `tradingCompetitionUser_getSwapContext` - 获取Swap组件所需的上下文信息 (指定网络、DEX、允许的交易对、默认Token等)。
    -   前端界面: 
        -   提供钱包连接/切换入口。
        -   集成通用的Swap组件。
        -   根据 `SwapContextDTO` 配置Swap组件，限定交易网络、DEX、可选Token。
        -   用户余额显示。
        -   执行Swap后，相关交易数据应由后端服务或数据管道捕获并处理，以更新用户交易量 (BR-TC-002, BR-TC-003)。
-   **涉及DTOs**: `SwapContextDTO`, `TokenInfoDTO`.
-   **数据库交互**: 读取 `trading_competitions`, `trading_competition_pools` 以确定Swap上下文。
-   **验收标准**:
    -   用户可以连接钱包。
    -   Swap组件能根据活动配置正确加载。
    -   用户可以执行Swap操作 (交易本身由钱包和DEX处理，此任务关注前端集成和上下文传递)。
-   **预计工作量**: 大 (Swap组件集成和配置复杂)

## 4. 活动数据展示 - 总览与我的数据 (task-tc-004)

-   **父任务**: task-tc-002
-   **描述**: 展示活动的总览统计数据以及当前连接钱包用户的个人参与数据。
    -   API: `tradingCompetitionUser_getCompetitionDetail` (部分复用) - 返回总奖池价值 (USD)、总有效交易量 (USD)、参与人数。
    -   API: `tradingCompetitionUser_getMyCompetitionData` - 获取当前用户在活动中各奖池的数据 (个人交易量、占比、排名、预估奖励)。
    -   前端界面:
        -   展示总览数据：总奖池价值, 总交易量, 参与人数。
        -   "我的数据"Tab/区域：
            -   支持按奖池切换查看。
            -   展示用户在各奖池的有效交易量、占比、排名、预计奖励。
-   **涉及DTOs**: `CompetitionDetailForUserDTO` (总览部分), `UserCompetitionOverallDataDTO`, `UserCompetitionDataDTO`.
-   **数据来源**: 总览数据可能来自 `trading_competitions` (聚合字段)；用户数据来自 `trading_competition_user_volume` (假设由数据管道填充) 或实时计算服务。
-   **验收标准**:
    -   页面能正确展示活动的总览数据。
    -   已连接钱包的用户能看到自己在各奖池的交易数据和排名信息。
-   **预计工作量**: 中 (依赖后端数据统计的准确性)

## 5. 活动数据展示 - Leaderboard (task-tc-005)

-   **父任务**: task-tc-002
-   **描述**: 展示活动各奖池的排行榜。
    -   API: `tradingCompetitionUser_getPoolLeaderboard` - 分页获取指定奖池的排行榜数据 (排名、用户地址、交易量、占比、预估奖励)，可选高亮当前用户。
    -   前端界面:
        -   "Leaderboard"Tab/区域。
        -   支持按奖池切换查看。
        -   表格展示排行榜，包含排名、用户地址/昵称、交易量、占比、预估奖励。
        -   支持分页或滚动加载。
        -   (可选MVP后) 支持用户搜索自己的地址/昵称。
-   **涉及DTOs**: `LeaderboardEntryDTO`.
-   **数据来源**: `trading_competition_user_volume` (假设由数据管道填充) 或实时计算服务。
-   **验收标准**:
    -   用户可以查看各奖池的Leaderboard。
    -   排行榜数据按规则排序，分页功能正常。
-   **预计工作量**: 中 (依赖后端数据统计的准确性)

## 6. 活动状态处理与展示 (task-tc-006)

-   **父任务**: task-tc-002
-   **描述**: 根据活动的不同时间状态 (未开始、进行中、已结束) 和用户状态 (未连接钱包) 显示不同的界面提示和功能。
    -   前端逻辑:
        -   **未开始**: 显示倒计时，Swap区域可能禁用或提示。
        -   **进行中**: `GetCompetitionDetail` API中的 `current_status` 和 `remaining_time_seconds` 会反映此状态，正常显示倒计时。
        -   **已结束 (结果计算中)**: `GetCompetitionDetail` API中的 `current_status` (如 `CALCULATING_RESULTS`)，前端显示相应提示，隐藏Swap，个人数据可能显示最终快照或提示计算中。
        -   **未连接钱包**: 提示连接钱包，Swap区域和个人数据区域禁用或隐藏。
-   **验收标准**:
    -   页面能根据活动状态和用户钱包连接状态展示正确的UI和提示信息。
-   **预计工作量**: 小

## 7. 活动结束 - 结果展示与奖励领取 (task-tc-007)

-   **父任务**: task-tc-001, task-tc-002
-   **描述**: 活动结束后，向用户展示其最终结果（获奖/未获奖），并为获奖用户提供奖励领取功能。
    -   API: `tradingCompetitionUser_getUserActivityResult` - 获取用户在活动结束后的结果 (总体状态、奖励列表、最终数据)。
    -   API: `tradingCompetitionUser_claimReward` - 用户发起领取单个奖励项。
    -   (可选API: `tradingCompetitionUser_claimAllRewards` - 用户一键领取所有可领奖励)。
    -   前端界面:
        -   **获奖用户**: 显示获奖提示，分奖池展示获得的奖励详情，提供 "Claim" (或 "Claim All") 按钮，显示奖励领取状态。
        -   **未获奖用户**: 显示未获奖提示，展示最终排名和交易量数据。
        -   后端逻辑 (`claimReward`): 校验奖励是否可领取，与 `taskon-actions` 或专门的奖励发放服务交互完成链上发放，更新 `trading_competition_user_rewards` 表中的 `claim_status`, `claimed_tx_hash`, `claimed_at`。
-   **涉及DTOs**: `UserActivityResultDTO`, `UserRewardItemDTO`, `ClaimRewardParamsDTO`, `ClaimRewardResponseDTO`.
-   **数据库交互**: `trading_competition_user_rewards` 表的查询和更新。
-   **验收标准**:
    -   用户在活动结束后能看到自己的最终结果（获奖/未获奖）。
    -   获奖用户能看到奖励详情，并成功发起领取操作。
    -   奖励领取状态能正确更新。
-   **预计工作量**: 大 (领奖流程涉及与actions服务或链交互)

## 8. 业务规则实现 - C端 (task-tc-008)

-   **父任务**: 分散在各个功能模块任务中
-   **描述**: 在C端功能中体现和校验相关的业务规则。
    -   BR-TC-001 (钱包连接要求) - 在task-tc-003, task-tc-004, task-tc-007中体现。
    -   BR-TC-007 (Leaderboard数据刷新频率) - 前端展示时可提示数据更新情况，实际刷新由API数据决定。
    -   BR-TC-008 (奖励领取期限) - API (`getUserActivityResult`) 返回的奖励项中应包含期限信息，前端展示并据此禁用过期奖励的领取按钮。
    -   其他规则如有效交易定义、交易量计算、奖池瓜分、获奖资格主要由后端或数据管道处理，C端负责展示结果。
-   **验收标准**:
    -   C端功能符合相关业务规则的展示和交互要求。
-   **预计工作量**: 小 (主要是展示和前端校验)

---
**说明:**
- 每个任务都应包含相应的单元测试和集成测试 (前端单元测试，E2E测试)。
- C端许多数据依赖B端配置和后端数据处理 (特别是交易量统计和奖励计算)。
- `taskon-actions` 或专门的奖励服务在奖励计算和发放环节扮演重要角色。
- "预计工作量"是相对估算。 