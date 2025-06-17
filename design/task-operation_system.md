# 运营体系 - 功能开发任务拆解

本文档基于 `新需求/整理后的md/new-运营体系需求文档.md` 及其对应的设计文档 `design/new_operation_system_design.md`，旨在将用户运营体系功能分解为可执行的开发任务。任务拆解以功能模块为主要粒度，目标是实现满足核心需求的最小化可行产品（MVP）。

## 1. 核心数据模型与基础架构 (task-os-001)

- **父任务**: 无
- **描述**:
  - 根据 `design/new_operation_system_design.md` 中的数据库设计，创建以下核心表结构：
    - `user_operation_profiles`: 用户运营档案表 (积分、经验、等级、签到信息)。
    - `level_configs`: 等级配置表。
    - `points_transactions`: 积分流水表。
    - `experience_transactions`: 经验流水表。
    - `daily_checkin_configs`: 每日签到配置表。
    - `daily_checkin_records`: 用户每日签到记录表。
    - `badges`: 勋章配置表。
    - `user_badges`: 用户勋章表。
    - `mall_products`: 积分商城商品表。
    - `mall_redemption_orders`: 积分商城兑换订单表。
  - 搭建项目基础结构，包括相关的C端DTO (opsuserdto) 和B端DTO (opsadmindto)，以及服务层骨架。
- **验收标准**:
  - 数据库表结构成功创建并通过迁移脚本管理。
  - 核心DTO定义完成。
  - 基础服务层接口和实现骨架搭建完成。
- **预计工作量**: 大

## 2. 用户运营档案管理 (task-os-002)

- **父任务**: task-os-001
- **描述**: 实现用户运营档案的创建、查询和基本更新。
  - C端API: `opsUser_getMyProfile` - 获取当前用户积分、经验、等级、签到信息。
  - 后端逻辑: 用户首次触发运营相关行为时（如首次签到），自动创建 `user_operation_profiles` 记录。
  - 系统内部更新：积分、经验、等级、签到信息变动时，更新 `user_operation_profiles`。
- **涉及DTOs**: `opsuserdto.UserOperationProfileDTO`.
- **数据库交互**: `user_operation_profiles` 表的增删改查。
- **验收标准**:
  - 用户档案能被正确创建和查询。
  - 用户积分、经验、等级、签到状态能通过API正确返回。
- **预计工作量**: 中

## 3. 等级系统 - B端配置 (task-os-003)

- **父任务**: task-os-001
- **描述**: 实现B端管理员对等级系统的配置功能。
  - B端API: `opsAdmin_upsertLevelConfig` - 创建或更新等级配置 (等级序号、名称、经验阈值、权益描述、图标、是否启用)。
  - B端API: `opsAdmin_listLevelConfigs` - 分页获取等级配置列表。
  - B端API: `opsAdmin_deleteLevelConfig` - 删除等级配置。
  - 后端逻辑: 校验等级配置的有效性（如等级序号唯一性）。
- **涉及DTOs**: `opsadmindto.LevelConfigParamsDTO`, `opsadmindto.LevelConfigDTO`.
- **数据库交互**: `level_configs` 表的增删改查。
- **验收标准**:
  - 管理员可以成功创建、编辑、查看和删除等级配置。
  - 等级配置信息能正确存储和展示。
- **预计工作量**: 中

## 4. 等级系统 - C端展示与自动升级 (task-os-004)

- **父任务**: task-os-002, task-os-003
- **描述**: 实现C端用户等级的展示和经验值累积自动升级逻辑。
  - C端API: `opsUser_getMyProfile` (复用) - 返回中包含当前等级信息和下一级所需经验。
  - 后端逻辑: 用户获得经验时，检查是否达到下一等级阈值，如达到则自动更新 `user_operation_profiles` 中的 `current_level_id`，并记录经验流水。发送等级提升通知 (task-os-013)。
- **涉及DTOs**: `opsuserdto.UserLevelInfoDTO` (嵌套在 `UserOperationProfileDTO` 中)。
- **数据库交互**: `user_operation_profiles` 表的更新, `experience_transactions` 表的创建。
- **验收标准**:
  - 用户等级和升级进度能在C端正确展示。
  - 用户经验值增加时，能自动判断并完成升级。
  - 升级后，用户档案中的等级信息更新。
- **预计工作量**: 中

## 5. 每日签到 - B端配置 (task-os-005)

- **父任务**: task-os-001
- **描述**: 实现B端管理员对每日签到规则的配置。
  - B端API: `opsAdmin_getDailyCheckinConfig` - 获取当前签到配置。
  - B端API: `opsAdmin_updateDailyCheckinConfig` - 更新签到配置 (基础积分/经验奖励、连续签到规则、补签开关、补签消耗、最大补签天数)。
  - 后端逻辑: 保存唯一的签到配置记录。校验配置有效性。
- **涉及DTOs**: `opsadmindto.DailyCheckinConfigDTO`.
- **数据库交互**: `daily_checkin_configs` 表的查询和更新。
- **验收标准**:
  - 管理员可以配置每日签到和连续签到的奖励规则。
  - 可以配置补签功能及其消耗。
  - 配置信息能正确保存和获取。
- **预计工作量**: 小

## 6. 每日签到 - C端功能 (task-os-006)

- **父任务**: task-os-002, task-os-005
- **描述**: 实现C端用户每日签到、查看签到状态和补签功能。
  - C端API: `opsUser_getCheckinStatus` - 获取用户今日是否签到、连续天数、签到日历、奖励规则、补签信息。
  - C端API: `opsUser_performCheckin` - 执行当日签到，发放奖励，记录签到。
  - C端API: `opsUser_performMissedCheckin` - 执行补签，扣除补签消耗，发放奖励，记录补签。
  - 后端逻辑: 处理签到/补签逻辑，更新用户积分/经验 (`user_operation_profiles`, `points_transactions`, `experience_transactions`)，记录 `daily_checkin_records`，更新连续签到天数。检查勋章触发 (task-os-008)。发送通知 (task-os-013)。
- **涉及DTOs**: `opsuserdto.CheckinStatusDTO`, `opsuserdto.PerformCheckinResponseDTO`.
- **数据库交互**: `daily_checkin_records` (创建), `user_operation_profiles` (更新), `points_transactions` (创建), `experience_transactions` (创建)。
- **验收标准**:
  - 用户可以成功进行每日签到并获得相应奖励。
  - 用户可以查看自己的签到日历和状态。
  - 如果配置允许，用户可以进行补签。
  - 签到和补签行为能正确更新用户数据和流水。
- **预计工作量**: 大

## 7. 勋章系统 - B端配置 (task-os-007)

- **父任务**: task-os-001
- **描述**: 实现B端管理员对勋章的配置功能。
  - B端API: `opsAdmin_upsertBadgeConfig` - 创建或更新勋章 (名称、描述、图标、稀有度、获取条件JSON、是否启用、显示顺序)。
  - B端API: `opsAdmin_listBadgeConfigs` - 分页获取勋章配置列表。
  - B端API: `opsAdmin_deleteBadgeConfig` - 删除勋章配置。
  - 后端逻辑: 校验勋章配置有效性 (如名称唯一性)。
- **涉及DTOs**: `opsadmindto.BadgeConfigParamsDTO`, `opsadmindto.BadgeConfigDTO`.
- **数据库交互**: `badges` 表的增删改查。
- **验收标准**:
  - 管理员可以成功创建、编辑、查看和删除勋章配置。
  - 勋章获取条件能以JSON格式存储。
- **预计工作量**: 中

## 8. 勋章系统 - C端获取与展示 (task-os-008)

- **父任务**: task-os-002, task-os-007
- **描述**: 实现C端用户勋章的自动授予、列表展示和佩戴功能。
  - C端API: `opsUser_listMyBadges` - 获取用户已获得和可选的未获得勋章列表。
  - C端API: `opsUser_wearBadge` - 用户选择佩戴某个勋章 (如果业务支持)。
  - 后端逻辑: 在特定用户行为发生后 (如签到、升级、完成任务等)，根据 `badges` 表中配置的 `acquisition_criteria_json` 判断是否满足勋章获取条件。满足则在 `user_badges` 表中创建记录。发送勋章获取通知 (task-os-013)。
- **涉及DTOs**: `opsuserdto.BadgeDTO`.
- **数据库交互**: `user_badges` 表的创建和更新, `badges` 表查询。
- **验收标准**:
  - 用户完成特定条件后能自动获得勋章。
  - 用户可以在C端查看自己已获得的勋章和未获得的勋章（及获取条件）。
  - （可选）用户可以佩戴勋章。
- **预计工作量**: 大 (获取条件判断逻辑可能复杂)

## 9. 积分商城 - B端商品配置 (task-os-009)

- **父任务**: task-os-001, task-os-003 (依赖等级配置)
- **描述**: 实现B端管理员对积分商城商品的配置。
  - B端API: `opsAdmin_upsertMallProductConfig` - 创建或更新商品 (名称、描述、类型、图片、积分成本、库存、用户限兑、总限兑、等级限制、有效期、是否上架、显示顺序)。
  - B端API: `opsAdmin_listMallProductConfigs` - 分页获取商品配置列表。
  - B端API: `opsAdmin_deleteMallProductConfig` - 删除商品配置。
- **涉及DTOs**: `opsadmindto.MallProductConfigParamsDTO`, `opsadmindto.MallProductConfigDTO`.
- **数据库交互**: `mall_products` 表的增删改查。
- **验收标准**:
  - 管理员可以成功创建、编辑、查看和删除商城商品。
  - 商品信息和兑换限制能正确配置。
- **预计工作量**: 中

## 10. 积分商城 - C端浏览与兑换 (task-os-010)

- **父任务**: task-os-002, task-os-009
- **描述**: 实现C端用户浏览积分商城商品、查看详情和使用积分兑换。
  - C端API: `opsUser_listMallProducts` - 获取商品列表 (支持分类、排序、分页)，返回项包含用户是否可兑换状态。
  - C端API: `opsUser_getMallProductDetail` - 获取商品详情，包含用户已兑换次数和是否可兑换。
  - C端API: `opsUser_redeemMallProduct` - 用户兑换商品。后端处理积分扣减、库存更新、订单生成、商品发放(虚拟)/状态变更(实体)。
  - 后端逻辑: 校验兑换条件 (积分是否足够、库存、等级限制、兑换次数限制、有效期)。创建 `mall_redemption_orders` 记录。更新 `user_operation_profiles` (积分)，`mall_products` (库存)。发送通知 (task-os-013)。
- **涉及DTOs**: `opsuserdto.MallProductListItemDTO`, `opsuserdto.MallProductDetailDTO`, `opsuserdto.RedeemMallProductResponseDTO`.
- **数据库交互**: `mall_products` (查询/更新库存), `user_operation_profiles` (更新积分), `points_transactions` (创建), `mall_redemption_orders` (创建)。
- **验收标准**:
  - 用户可以浏览商品列表和详情。
  - 用户可以成功兑换符合条件的商品。
  - 积分、库存、订单状态等能正确更新。
  - 不符合条件的兑换请求被拒绝。
- **预计工作量**: 大

## 11. 流水与记录查询 - C端 (task-os-011)

- **父任务**: task-os-002, task-os-006, task-os-010
- **描述**: 实现C端用户查询自己的积分流水、经验流水和商城兑换记录。
  - C端API: `opsUser_listMyRedemptionOrders` - 分页获取用户的兑换记录。
  - C端API: `opsUser_listMyPointsTransactions` - 分页获取用户的积分流水。
  - C端API: `opsUser_listMyExperienceTransactions` - 分页获取用户的经验流水。
- **涉及DTOs**: `opsuserdto.RedemptionOrderDTO`, `opsuserdto.PointsTransactionDTO`, `opsuserdto.ExperienceTransactionDTO`.
- **数据库交互**: `mall_redemption_orders`, `points_transactions`, `experience_transactions` 表的查询。
- **验收标准**:
  - 用户可以查看自己的积分、经验流水和商城兑换订单。
  - 列表支持分页。
- **预计工作量**: 中

## 12. 管理员操作 - B端 (task-os-012)

- **父任务**: task-os-001
- **描述**: 实现B端管理员对用户资源调整、勋章授予和订单管理的功能。
  - B端API: `opsAdmin_listUserRedemptionOrders` - 管理员查看用户兑换订单列表 (支持筛选)。
  - B端API: `opsAdmin_updateRedemptionOrderStatus` - 管理员更新兑换订单状态 (如发货后标记完成)。
  - B端API: `opsAdmin_adjustUserResources` - 管理员手动调整用户积分/经验 (用于补偿或处理作弊)。
  - B端API: `opsAdmin_grantBadgeToUser` - 管理员手动授予用户勋章。
  - 后端逻辑: 记录管理员操作日志。调整资源时创建相应流水。
- **涉及DTOs**: `opsadmindto.AdminAdjustUserResourceParamsDTO`, `opsadmindto.AdminGrantUserBadgeParamsDTO`, `opsuserdto.RedemptionOrderDTO` (for listing).
- **数据库交互**: `mall_redemption_orders` (查询/更新), `user_operation_profiles` (更新), `points_transactions` (创建), `experience_transactions` (创建), `user_badges` (创建)。
- **验收标准**:
  - 管理员可以查询和管理用户兑换订单。
  - 管理员可以手动调整用户的积分和经验。
  - 管理员可以手动为用户授予勋章。
  - 操作有相应记录。
- **预计工作量**: 中

## 13. 通知系统集成 (task-os-013)

- **父任务**: task-os-004, task-os-006, task-os-008, task-os-010
- **描述**: 在运营体系的关键事件发生时，通过平台现有的通知系统向用户发送通知。
  - 事件包括：等级提升、获得勋章、积分商城兑换成功/失败、签到获得大奖（如果设计）。
  - 后端逻辑: 在上述事件的业务逻辑处理完成后，调用通知服务接口发送通知。
- **验收标准**:
  - 用户在等级提升、获得勋章、商城兑换等关键节点能收到平台通知。
- **预计工作量**: 小 (依赖现有通知系统)

## 14. 业务规则实现 (task-os-014)

- **父任务**: 分散在各个功能模块任务中
- **描述**: 在各模块的后端逻辑中，实现需求文档中定义的业务规则。
  - BR-OS-001 (积分获取规则) - 在签到、任务（假设外部）、邀请（假设外部）模块中实现。
  - BR-OS-002 (经验获取规则) - 在签到、任务（假设外部）模块中实现。
  - BR-OS-003 (积分有效期) - MVP阶段可选，若实现需定时任务和用户档案扩展。
  - BR-OS-004 (等级只升不降) - 已在task-os-004中体现。
  - BR-OS-005 (等级权益叠加) - 配置层面和C端展示逻辑。
  - BR-OS-006 (勋章获取唯一性) - 在task-os-008勋章授予逻辑中判断。
  - BR-OS-007 (商城兑换限制) - 在task-os-010兑换逻辑中校验。
  - BR-OS-008 (积分经验数值范围) - DTO和数据库设计层面体现。
  - BR-OS-009 (签到中断处理) - 在task-os-006签到逻辑中处理连续签到计数。
  - BR-OS-010 (作弊行为处理) - 主要依赖task-os-012管理员调整功能，监控机制为额外任务。
- **验收标准**:
  - 所有列出的核心业务规则在相关操作中得到正确执行和校验。
- **预计工作量**: 中 (分散实现)

---

**说明:**

- 每个任务都应包含相应的单元测试和集成测试。
- 前端任务与后端API开发紧密相关，建议并行或紧随其后进行。
- 任务系统和邀请系统的具体实现不包含在此次运营体系的核心MVP任务中，但运营体系需要能响应这些系统产生的积分/经验奖励（例如通过MQ消息或内部调用）。
- "预计工作量"是相对估算。
