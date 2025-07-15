# 交易大赛 - B端功能设计文档

## 1. 功能概述

本文档根据 `新需求/整理后的md/new-交易大赛-B端.md` 设计交易大赛B端（项目方）管理功能。项目方可以通过本模块提供的API接口，实现交易大赛活动的创建、详细配置（包括活动信息、参与项目、多样的奖池规则、页面视觉样式）、预算管理与充值、活动的发布与状态管理，以及活动上线后的数据监控与分析（包括汇总数据、排行榜、交易明细）。

该设计旨在提供一套完整、灵活且易于集成的B端解决方案，支持项目方高效地发起和运营链上交易竞赛，以提升用户参与度和代币活跃度。

## 2. 数据库设计 (MySQL)

为支持B端交易大赛管理功能，设计以下核心数据表。这些表将存储活动配置、参与项目、奖池规则、页面样式、预算资金以及用于数据分析的基础数据。

```sql
-- trading_competitions: 交易大赛主表，存储活动的核心信息
CREATE TABLE `trading_competitions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `creator_user_id` BIGINT UNSIGNED NOT NULL COMMENT '创建活动的B端用户ID (关联用户表)',
  `name` VARCHAR(100) NOT NULL COMMENT '活动名称, B端可自定义, 最多100字符',
  `start_time` DATETIME(3) NOT NULL COMMENT '活动开始时间 (UTC)',
  `end_time` DATETIME(3) NOT NULL COMMENT '活动结束时间 (UTC)',
  `banner_pc_url` VARCHAR(512) DEFAULT NULL COMMENT 'PC端Banner图片URL',
  `banner_mobile_url` VARCHAR(512) DEFAULT NULL COMMENT 'Mobile端Banner图片URL',
  `dot_animation_enabled` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否启用点状背景动画',
  `global_network_id` VARCHAR(50) NOT NULL COMMENT '全局交易网络ID (例如 "1" for Ethereum, "56" for BSC)',
  `global_dex_id` VARCHAR(100) NOT NULL COMMENT '全局DEX ID (例如 "uniswap_v3", "taskon_aggregator")',
  `page_style_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '关联的页面样式ID (外键, trading_competition_page_styles.id)',
  `status` VARCHAR(30) NOT NULL DEFAULT 'DRAFT' COMMENT '活动状态 (DRAFT:草稿, UPCOMING:即将开始, ONGOING:进行中, ENDED:已结束, CALCULATING_RESULTS:结果计算中, RESULTS_ANNOUNCED:结果已公布)',
  `total_budget_usd_estimate` DECIMAL(20, 8) DEFAULT NULL COMMENT '预估总奖池价值 (USD, 根据配置时的代币价格估算)',
  `total_funded_usd` DECIMAL(20, 8) DEFAULT 0.00 COMMENT '已充值总金额的USD价值 (根据充值时的代币价格计算)',
  `publish_validation_notes` TEXT DEFAULT NULL COMMENT '发布校验备注 (例如：预算不足、未设置奖池等)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_creator_user_id` (`creator_user_id`),
  INDEX `idx_status_start_time` (`status`, `start_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛活动主表';

-- trading_competition_page_styles: 交易大赛C端页面样式配置
CREATE TABLE `trading_competition_page_styles` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `background_type` VARCHAR(20) DEFAULT 'COLOR' COMMENT '背景类型 (COLOR:纯色, IMAGE:图片)',
  `background_value` VARCHAR(512) DEFAULT NULL COMMENT '背景值 (颜色代码或图片URL)',
  `theme_color` VARCHAR(20) DEFAULT NULL COMMENT '主题色 (例如十六进制颜色代码)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_competition_id` (`competition_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛页面样式配置表';

-- trading_competition_projects: 交易大赛参与项目信息
CREATE TABLE `trading_competition_projects` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `project_name` VARCHAR(100) NOT NULL COMMENT '项目名称',
  `project_logo_url` VARCHAR(512) DEFAULT NULL COMMENT '项目Logo URL',
  `project_twitter_handle` VARCHAR(100) DEFAULT NULL COMMENT '项目Twitter Handle (用于自动抓取信息)',
  `project_website_url` VARCHAR(512) DEFAULT NULL COMMENT '项目官网URL (可选)',
  `display_order` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '显示顺序，越小越靠前',
  `is_host_project` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否为活动主办方项目 (默认第一个添加的为host)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_competition_id_order` (`competition_id`, `display_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛参与项目表';

-- trading_competition_pools: 交易大赛奖池配置
CREATE TABLE `trading_competition_pools` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `pool_name` VARCHAR(100) NOT NULL COMMENT '奖池名称 (例如 Basic Pool, XXX Token Reward Pool)',
  `pool_type` VARCHAR(20) NOT NULL COMMENT '奖池类型 (BASIC:基础奖池, SPECIFIED_REWARD:指定代币奖池)',
  `is_enabled` BOOLEAN NOT NULL DEFAULT TRUE COMMENT '奖池是否启用 (主要用于多项目方时Basic Pool的可选开关)',
  `reward_token_address` VARCHAR(255) NOT NULL COMMENT '奖励代币的合约地址',
  `reward_token_symbol` VARCHAR(50) NOT NULL COMMENT '奖励代币的符号',
  `reward_token_decimals` INT UNSIGNED NOT NULL COMMENT '奖励代币的精度',
  `reward_token_logo_url` VARCHAR(512) DEFAULT NULL COMMENT '奖励代币的Logo URL',
  `reward_amount_raw` VARCHAR(78) NOT NULL COMMENT '奖励数量 (以原始精度的字符串形式存储, 支持大数)',
  `reward_amount_usd_estimate` DECIMAL(20, 8) DEFAULT NULL COMMENT '奖励金额的USD估值 (配置时估算)',
  `swap_from_type` VARCHAR(30) NOT NULL DEFAULT 'ANY_TOKEN' COMMENT 'Swap交易源代币类型 (ANY_TOKEN:任意代币, SPECIFIED_TOKENS:指定代币列表)',
  `swap_from_tokens_json` JSON DEFAULT NULL COMMENT '源代币列表 (当type为SPECIFIED_TOKENS时, 存储代币地址数组: [{"address":"0x...", "symbol":"USDT"}, ...])',
  `swap_to_type` VARCHAR(30) NOT NULL DEFAULT 'ANY_TOKEN' COMMENT 'Swap交易目标代币类型 (ANY_TOKEN:任意代币, SPECIFIED_TOKENS:指定代币列表, SAME_AS_REWARD_TOKEN:同奖励代币)',
  `swap_to_tokens_json` JSON DEFAULT NULL COMMENT '目标代币列表 (当type为SPECIFIED_TOKENS时, 存储代币地址数组)',
  `exclude_same_underlying_assets` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否排除同底层资产的交易对 (例如USDT/USDC, 仅当SwapFrom/To均为ANY_TOKEN时有效)',
  `winner_eligibility_enabled` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否启用获奖资格限制',
  `winner_eligibility_type` VARCHAR(30) DEFAULT NULL COMMENT '获奖资格类型 (MIN_VOLUME_USD:最低交易额, TOP_RANK:排行榜名次)',
  `winner_eligibility_min_volume_usd` DECIMAL(20, 8) DEFAULT NULL COMMENT '最低交易额门槛 (USD)',
  `winner_eligibility_top_rank` INT UNSIGNED DEFAULT NULL COMMENT '排行榜名次上限',
  `display_order` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '奖池显示顺序 (影响C端和B端数据分析Tab顺序)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_competition_id_order` (`competition_id`, `display_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛奖池配置表';

-- trading_competition_budget_deposits: 交易大赛预算充值记录
CREATE TABLE `trading_competition_budget_deposits` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `depositor_user_id` BIGINT UNSIGNED NOT NULL COMMENT '充值操作的B端用户ID',
  `token_address` VARCHAR(255) NOT NULL COMMENT '充值代币的合约地址',
  `token_symbol` VARCHAR(50) NOT NULL COMMENT '充值代币的符号',
  `token_decimals` INT UNSIGNED NOT NULL COMMENT '充值代币的精度',
  `deposited_amount_raw` VARCHAR(78) NOT NULL COMMENT '充值数量 (原始精度字符串)',
  `deposited_amount_usd_value` DECIMAL(20, 8) DEFAULT NULL COMMENT '充值金额当时的USD价值',
  `deposit_network_id` VARCHAR(50) NOT NULL COMMENT '充值交易所在的网络ID',
  `deposit_tx_hash` VARCHAR(255) NOT NULL COMMENT '充值交易哈希',
  `deposited_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '充值时间',
  `notes` VARCHAR(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  INDEX `idx_competition_id_token` (`competition_id`, `token_address`),
  UNIQUE KEY `uk_tx_hash_network` (`deposit_tx_hash`, `deposit_network_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛预算充值记录表';

-- trading_competition_user_volume: 用户在各奖池的交易量和次数统计 (由数据管道更新)
-- 此表也用于C端展示个人数据和B端排行榜数据源
CREATE TABLE `trading_competition_user_volume` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID',
  `pool_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的奖池ID',
  `user_wallet_address` VARCHAR(255) NOT NULL COMMENT '用户钱包地址',
  `total_trading_volume_usd` DECIMAL(30, 8) NOT NULL DEFAULT 0.00 COMMENT '累计有效交易量 (USD计价)',
  `total_transaction_count` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '累计有效交易笔数',
  `first_trade_at` DATETIME(3) DEFAULT NULL COMMENT '首次有效交易时间',
  `last_trade_at` DATETIME(3) DEFAULT NULL COMMENT '最后一次有效交易时间',
  `rank_in_pool` INT UNSIGNED DEFAULT NULL COMMENT '在奖池中的排名 (定期更新)',
  `last_calculated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '本条记录数据最后计算更新的时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_pool_user` (`pool_id`, `user_wallet_address`),
  INDEX `idx_competition_user` (`competition_id`, `user_wallet_address`),
  INDEX `idx_pool_volume_rank` (`pool_id`, `total_trading_volume_usd` DESC, `rank_in_pool` ASC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户在奖池的交易量统计表';

-- trading_competition_daily_summary: 交易大赛每日汇总数据 (B端数据分析用，由数据管道更新)
CREATE TABLE `trading_competition_daily_summary` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID',
  `pool_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '关联的奖池ID (NULL表示整个活动的汇总)',
  `summary_date` DATE NOT NULL COMMENT '汇总日期',
  `daily_volume_usd` DECIMAL(30, 8) NOT NULL DEFAULT 0.00 COMMENT '当日总交易量 (USD)',
  `daily_new_participants` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '当日新增参与用户数',
  `daily_transactions` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '当日总交易笔数',
  `cumulative_volume_usd` DECIMAL(30, 8) NOT NULL DEFAULT 0.00 COMMENT '截止当日累计总交易量 (USD)',
  `cumulative_participants` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '截止当日累计参与用户数',
  `cumulative_transactions` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '截止当日累计交易笔数',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_comp_pool_date` (`competition_id`, `pool_id`, `summary_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛每日汇总数据表';

-- trading_competition_raw_transactions: 记录符合条件的原始交易数据 (B端明细查看, 可能由数据管道填充部分或全部)
-- 注意: 对于海量数据，可能考虑专用数据仓库。此处为简化设计。
CREATE TABLE `trading_competition_raw_transactions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID',
  `pool_ids_json` JSON DEFAULT NULL COMMENT '该交易贡献的奖池ID列表 (JSON数组, 因为一笔交易可能满足多个奖池的条件)',
  `user_wallet_address` VARCHAR(255) NOT NULL COMMENT '用户钱包地址',
  `transaction_hash` VARCHAR(255) NOT NULL COMMENT '交易哈希',
  `network_id` VARCHAR(50) NOT NULL COMMENT '交易所在网络ID',
  `block_timestamp` DATETIME(3) NOT NULL COMMENT '区块时间戳',
  `from_token_address` VARCHAR(255) NOT NULL COMMENT '源代币合约地址',
  `from_token_symbol` VARCHAR(50) DEFAULT NULL COMMENT '源代币符号',
  `from_token_amount_raw` VARCHAR(78) NOT NULL COMMENT '源代币数量 (原始精度)',
  `to_token_address` VARCHAR(255) NOT NULL COMMENT '目标代币合约地址',
  `to_token_symbol` VARCHAR(50) DEFAULT NULL COMMENT '目标代币符号',
  `to_token_amount_raw` VARCHAR(78) NOT NULL COMMENT '目标代币数量 (原始精度)',
  `transaction_value_usd` DECIMAL(20, 8) DEFAULT NULL COMMENT '该笔交易的USD价值',
  `dex_id` VARCHAR(100) DEFAULT NULL COMMENT '完成交易的DEX ID',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录插入时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tx_hash_network` (`transaction_hash`, `network_id`),
  INDEX `idx_competition_user_time` (`competition_id`, `user_wallet_address`, `block_timestamp` DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛原始交易明细表';

```

**表关系说明:**

- `trading_competitions` 是核心，其他表如 `trading_competition_page_styles`, `trading_competition_projects`, `trading_competition_pools`, `trading_competition_budget_deposits` 都通过 `competition_id` 与其关联。
- `trading_competition_page_styles` 与 `trading_competitions` 通过 `competition_id` 建立一对一关联（通过 UNIQUE KEY 实现，也可以将样式字段直接放入主表，但分离更清晰）。
- 数据分析相关的表 (`trading_competition_user_volume`, `trading_competition_daily_summary`, `trading_competition_raw_transactions`) 也与 `trading_competitions` 和可能的 `trading_competition_pools` 关联，它们通常由后台数据处理服务填充。
- 用户相关的ID (`creator_user_id`, `depositor_user_id`) 应关联到项目方的用户账户表（未在此处定义）。

此数据库设计旨在满足B端交易大赛管理的主要需求，包括灵活的活动配置、奖池设置、预算管理和基本的数据分析支持。

## 3. API 设计 (JSON-RPC)

本节定义了B端项目方管理交易大赛所需的JSON-RPC接口。设计遵循 `taskon-server` (或相关微服务) 的通用API规范，并力求与C端API在风格和通用数据结构上保持一致性。

**通用约定:**
- DTOs (Data Transfer Objects) 定义在 `internal/application/dto/` 或类似路径下的 `trading_competition_admin_dto.go` (示例文件名) 或相关文件中。
- 所有涉及金额、数量的原始精度数据均使用字符串类型以避免精度丢失。
- 日期和时间均使用 ISO8601 UTC 字符串格式。
- 关键业务操作的API会引用其需求来源文档章节。

### 3.1 DTOs (B-端管理)

```go
package dto // 假设在 internal/application/dto/ 或类似路径

import (
	"taskon/common" // 假设 common.Page, common.ListResponse, common.BaseSuccessResponse 在此
)

// TradingCompetitionBasicInfoParamsDTO 活动基本信息参数 (创建/更新)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 活动信息配置"
type TradingCompetitionBasicInfoParamsDTO struct {
	Name                string  `json:"name" binding:"required,max=100"` // 活动名称, 必填, 最多100字符
	StartTime           string  `json:"start_time" binding:"required"`    // 活动开始时间 (ISO8601 UTC), 必填
	EndTime             string  `json:"end_time" binding:"required"`      // 活动结束时间 (ISO8601 UTC), 必填
	BannerPCURL         *string `json:"banner_pc_url,omitempty"`          // PC端Banner图片URL
	BannerMobileURL     *string `json:"banner_mobile_url,omitempty"`      // Mobile端Banner图片URL
	DotAnimationEnabled *bool   `json:"dot_animation_enabled,omitempty"`  // 是否启用点状动画, 默认为false
	GlobalNetworkID     string  `json:"global_network_id" binding:"required"` // 全局交易网络ID, 必填
	GlobalDexID         string  `json:"global_dex_id" binding:"required"`     // 全局DEX ID, 必填
}

// TradingCompetitionPageStyleParamsDTO 页面样式配置参数 (创建/更新)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 页面样式配置"
type TradingCompetitionPageStyleParamsDTO struct {
	BackgroundType *string `json:"background_type,omitempty"` // 背景类型 (COLOR, IMAGE), 默认为COLOR
	BackgroundValue *string `json:"background_value,omitempty"` // 背景值 (颜色代码或图片URL)
	ThemeColor     *string `json:"theme_color,omitempty"`      // 主题色 (十六进制颜色代码)
}

// TradingCompetitionProjectParamsDTO 参与项目配置参数 (创建/更新)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 项目信息配置"
type TradingCompetitionProjectParamsDTO struct {
	ProjectName          string  `json:"project_name" binding:"required,max=100"` // 项目名称, 必填
	ProjectLogoURL       *string `json:"project_logo_url,omitempty"`               // 项目Logo URL
	ProjectTwitterHandle *string `json:"project_twitter_handle,omitempty"`         // 项目Twitter Handle
	ProjectWebsiteURL    *string `json:"project_website_url,omitempty"`            // 项目官网URL (可选)
	DisplayOrder         *int    `json:"display_order,omitempty"`                  // 显示顺序, 默认为0
	IsHostProject        *bool   `json:"is_host_project,omitempty"`                // 是否为活动主办方项目
}

// TokenIdentifierDTO 用于标识一个代币
type TokenIdentifierDTO struct {
	Address  string `json:"address" binding:"required"` // 代币合约地址
	Symbol   string `json:"symbol" binding:"required"`  // 代币符号
	Decimals int    `json:"decimals" binding:"required"` // 代币精度
	LogoURL  *string `json:"logo_url,omitempty"`        // 代币Logo URL (可选)
}

// TradingCompetitionPoolWinnerEligibilityParamsDTO 奖池获奖资格参数
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 奖励设置"
type TradingCompetitionPoolWinnerEligibilityParamsDTO struct {
	Enabled      bool    `json:"enabled"`                                  // 是否启用获奖资格限制, 默认false
	Type         *string `json:"type,omitempty"`                           // 获奖资格类型 (MIN_VOLUME_USD, TOP_RANK)
	MinVolumeUSD *string `json:"min_volume_usd,omitempty"`                 // 最低交易额门槛 (USD, 字符串表示的Decimal)
	TopRank      *int    `json:"top_rank,omitempty"`                       // 排行榜名次上限
}

// TradingCompetitionPoolParamsDTO 奖池配置参数 (创建/更新)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 奖励设置"
type TradingCompetitionPoolParamsDTO struct {
	PoolName                       string                                           `json:"pool_name" binding:"required,max=100"` // 奖池名称, 必填
	PoolType                       string                                           `json:"pool_type" binding:"required"`          // 奖池类型 (BASIC, SPECIFIED_REWARD), 必填
	IsEnabled                      *bool                                            `json:"is_enabled,omitempty"`                 // 奖池是否启用, 默认true (用于Basic Pool的可选开关)
	RewardToken                    TokenIdentifierDTO                               `json:"reward_token" binding:"required"`      // 奖励代币信息, 必填
	RewardAmountRaw                string                                           `json:"reward_amount_raw" binding:"required"`  // 奖励数量 (原始精度字符串), 必填
	SwapFromType                   string                                           `json:"swap_from_type" binding:"required"`    // Swap交易源代币类型 (ANY_TOKEN, SPECIFIED_TOKENS), 必填
	SwapFromTokens                 []TokenIdentifierDTO                             `json:"swap_from_tokens,omitempty"`           // 源代币列表 (当type为SPECIFIED_TOKENS时)
	SwapToType                     string                                           `json:"swap_to_type" binding:"required"`      // Swap交易目标代币类型 (ANY_TOKEN, SPECIFIED_TOKENS, SAME_AS_REWARD_TOKEN), 必填
	SwapToTokens                   []TokenIdentifierDTO                             `json:"swap_to_tokens,omitempty"`             // 目标代币列表 (当type为SPECIFIED_TOKENS时)
	ExcludeSameUnderlyingAssets    *bool                                            `json:"exclude_same_underlying_assets,omitempty"` // 是否排除同底层资产的交易对, 默认false
	WinnerEligibility              TradingCompetitionPoolWinnerEligibilityParamsDTO `json:"winner_eligibility"`                   // 获奖资格配置
	DisplayOrder                   *int                                             `json:"display_order,omitempty"`              // 显示顺序, 默认为0
	// PoolID is not needed for create, but for update it might be passed in URL or body
	PoolIDForUpdate *string `json:"pool_id_for_update,omitempty"` // 更新时用于标识奖池的ID (如果不在URL中传递)
}

// CreateTradingCompetitionParamsDTO 创建完整交易大赛参数 (包含所有子配置)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 创建活动"
type CreateTradingCompetitionParamsDTO struct {
	BasicInfo TradingCompetitionBasicInfoParamsDTO   `json:"basic_info" binding:"required"` // 活动基本信息
	Projects  []TradingCompetitionProjectParamsDTO `json:"projects" binding:"required,min=1"` // 参与项目列表, 至少一个
	Pools     []TradingCompetitionPoolParamsDTO    `json:"pools" binding:"required,min=1"`    // 奖池配置列表, 至少一个 (发布前校验)
	PageStyle *TradingCompetitionPageStyleParamsDTO `json:"page_style,omitempty"`           // 页面样式配置 (可选)
}

// UpdateTradingCompetitionParamsDTO 更新交易大赛参数 (各部分可选更新)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 编辑活动"
type UpdateTradingCompetitionParamsDTO struct {
	CompetitionID string                                   `json:"competition_id" binding:"required"` // 要更新的活动ID
	BasicInfo     *TradingCompetitionBasicInfoParamsDTO  `json:"basic_info,omitempty"`              // 活动基本信息 (可选更新)
	Projects      *[]TradingCompetitionProjectParamsDTO  `json:"projects,omitempty"`                // 参与项目列表 (可选更新，全量替换)
	Pools         *[]TradingCompetitionPoolParamsDTO     `json:"pools,omitempty"`                   // 奖池配置列表 (可选更新，全量替换或更细致操作)
	PageStyle     *TradingCompetitionPageStyleParamsDTO  `json:"page_style,omitempty"`              // 页面样式配置 (可选更新)
}

// TradingCompetitionListItemDTO 活动列表项信息 (B端管理视角)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 查看活动列表"
type TradingCompetitionListItemDTO struct {
	ID                        string  `json:"id"`                                 // 活动ID (数字字符串)
	Name                      string  `json:"name"`                               // 活动名称
	StartTime                 string  `json:"start_time"`                         // 活动开始时间 (ISO8601 UTC)
	EndTime                   string  `json:"end_time"`                           // 活动结束时间 (ISO8601 UTC)
	Status                    string  `json:"status"`                             // 活动状态 (DRAFT, UPCOMING, ONGOING, ENDED, ...)
	TotalBudgetUSDStr         *string `json:"total_budget_usd_str,omitempty"`     // 总预算金额 (USD, 格式化字符串)
	TotalFundedUSDStr         *string `json:"total_funded_usd_str,omitempty"`     // 已充值金额 (USD, 格式化字符串)
	CanEdit                   bool    `json:"can_edit"`                           // 是否可编辑
	CanDelete                 bool    `json:"can_delete"`                         // 是否可删除
	CanViewData               bool    `json:"can_view_data"`                      // 是否可查看数据分析
	NeedsFunding              bool    `json:"needs_funding"`                      // 是否需要充值 (预算不足)
	PublishValidationNotes    *string `json:"publish_validation_notes,omitempty"` // 发布校验问题备注
	CreatedAt                 string  `json:"created_at"`                         // 创建时间 (ISO8601 UTC)
}

// TradingCompetitionDetailAdminDTO 活动详情 (B端管理视角，包含所有配置)
// Requirement Source: Implied by Edit and View operations
type TradingCompetitionDetailAdminDTO struct {
	ID                         string                               `json:"id"`                           // 活动ID
	BasicInfo                  TradingCompetitionBasicInfoParamsDTO   `json:"basic_info"`                    // 活动基本信息
	Projects                   []TradingCompetitionProjectParamsDTO `json:"projects"`                      // 参与项目列表
	Pools                      []TradingCompetitionPoolParamsDTO    `json:"pools"`                         // 奖池配置列表
	PageStyle                  *TradingCompetitionPageStyleParamsDTO `json:"page_style,omitempty"`            // 页面样式配置
	Status                     string                               `json:"status"`                        // 活动状态
	TotalBudgetUSDStr          *string                              `json:"total_budget_usd_str,omitempty"`  // 总预算金额 (USD, 格式化字符串)
	TotalFundedUSDStr          *string                              `json:"total_funded_usd_str,omitempty"`  // 已充值金额 (USD, 格式化字符串)
	EstimatedFundedPercent     *float64                             `json:"estimated_funded_percent,omitempty"` // 预算完成百分比 (0-1.0)
	CreatorUserID              string                               `json:"creator_user_id"`               // 创建者用户ID
	CreatedAt                  string                               `json:"created_at"`                    // 创建时间
	UpdatedAt                  string                               `json:"updated_at"`                    // 最后更新时间
	PublishValidationNotes     *string                              `json:"publish_validation_notes,omitempty"`// 发布校验问题备注
}

// FetchProjectInfoByTwitterParamsDTO 通过Twitter Handle获取项目信息参数
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"
type FetchProjectInfoByTwitterParamsDTO struct {
	TwitterHandle string `json:"twitter_handle" binding:"required"` // 项目的Twitter Handle
}

// FetchedProjectInfoDTO 通过Twitter Handle获取到的项目信息
type FetchedProjectInfoDTO struct {
	ProjectName    string  `json:"project_name,omitempty"`     // 项目名称
	ProjectLogoURL *string `json:"project_logo_url,omitempty"` // 项目Logo URL
	// Potentially more fields like description, website etc.
}

// CheckProjectTokenParamsDTO 检查项目是否已发币参数
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"
type CheckProjectTokenParamsDTO struct {
	ProjectName        string  `json:"project_name,omitempty"`         // 项目名称 (用于在CMC/Alpha库中查询)
	ProjectTwitterHandle *string `json:"project_twitter_handle,omitempty"` // 或Twitter Handle
	// Could also accept contract address if known
}

// ProjectTokenCheckResultDTO 项目发币检查结果
type ProjectTokenCheckResultDTO struct {
	HasLaunchedToken bool                `json:"has_launched_token"`        // 是否已发币
	Token            *TokenIdentifierDTO `json:"token,omitempty"`             // 如果已发币，返回代币信息 (可选)
	Suggestion       *string             `json:"suggestion,omitempty"`        // 给B端的建议，例如 "建议为您的代币XXX创建一个专属奖池以提升交易量"
}

// DepositTokenParamsDTO B端充值Token参数
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：Token充值"
type DepositTokenParamsDTO struct {
	CompetitionID       string             `json:"competition_id" binding:"required"` // 目标活动ID
	Token               TokenIdentifierDTO `json:"token" binding:"required"`          // 要充值的代币信息
	AmountRaw           string             `json:"amount_raw" binding:"required"`    // 充值数量 (原始精度字符串)
	DepositNetworkID    string             `json:"deposit_network_id" binding:"required"` // 充值交易发生的网络ID
	DepositTxHash       string             `json:"deposit_tx_hash" binding:"required"`   // 充值交易哈希 (用于后台验证和记录)
}

// CompetitionBudgetStatusDTO 活动预算状态
// Requirement Source: Implied by Token充值 and Publish校验
type CompetitionBudgetStatusDTO struct {
	CompetitionID           string  `json:"competition_id"`                     // 活动ID
	TotalBudgetUSDStr       string  `json:"total_budget_usd_str"`             // 活动总预算需求 (USD, 格式化字符串)
	TotalFundedUSDStr       string  `json:"total_funded_usd_str"`             // 当前已充值总额 (USD, 格式化字符串)
	RemainingNeededUSDStr   string  `json:"remaining_needed_usd_str"`         // 仍需充值金额 (USD, 格式化字符串)
	IsSufficient            bool    `json:"is_sufficient"`                    // 预算是否充足
	DepositedTokens         []DepositedTokenInfoDTO `json:"deposited_tokens,omitempty"` // 已充值代币详情列表 (可选)
}

// DepositedTokenInfoDTO 已充值代币信息
type DepositedTokenInfoDTO struct {
	TokenSymbol      string `json:"token_symbol"`         // 代币符号
	DepositedAmountRaw string `json:"deposited_amount_raw"` // 已充值数量 (原始精度)
	DepositedValueUSDStr string `json:"deposited_value_usd_str"` // 已充值部分的USD价值 (格式化字符串)
}

// CompetitionDataRequestParamsDTO 数据分析通用请求参数 (分页、筛选)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 数据分析模块"
type CompetitionDataRequestParamsDTO struct {
	CompetitionID string      `json:"competition_id" binding:"required"`
	PoolID        *string     `json:"pool_id,omitempty"` // 可选，如果查询特定奖池数据，否则为总览
	Pagination    common.Page `json:"pagination"`
	// DateRange, Filters etc. can be added here
	DateRangeStart *string `json:"date_range_start,omitempty"` // ISO8601, for trend charts
	DateRangeEnd   *string `json:"date_range_end,omitempty"`   // ISO8601, for trend charts
}

// CompetitionSummaryMetricsDTO 汇总数据指标
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - 汇总数据"
type CompetitionSummaryMetricsDTO struct {
	TotalVolumeUSDStr      string  `json:"total_volume_usd_str"`        // 总交易量 (USD, 格式化)
	TotalParticipants      int     `json:"total_participants"`          // 总参与人数
	TotalTransactions      int     `json:"total_transactions"`          // 总交易笔数
	VolumeChangeRateStr    *string `json:"volume_change_rate_str,omitempty"` // 交易量日环比 (例如 "+5.2%")
	ParticipantsChangeRateStr *string `json:"participants_change_rate_str,omitempty"` // 参与人数日环比
	TransactionsChangeRateStr *string `json:"transactions_change_rate_str,omitempty"` // 交易笔数日环比
}

// TimeSeriesDataPointDTO 时间序列数据点
type TimeSeriesDataPointDTO struct {
	Timestamp string `json:"timestamp"` // 时间点 (ISO8601 Date or DateTime)
	Value     string `json:"value"`     // 数值 (字符串，可以是金额或计数)
	Value2    *string `json:"value2,omitempty"` // 第二个Y轴的数值 (可选, 例如用于参与人数和交易笔数同图)
}

// CompetitionDataSummaryDTO 数据分析汇总响应
type CompetitionDataSummaryDTO struct {
	Metrics           CompetitionSummaryMetricsDTO `json:"metrics"`                      // 核心指标
	VolumeTrend       []TimeSeriesDataPointDTO     `json:"volume_trend,omitempty"`       // 交易量趋势图数据
	ParticipantsTrend []TimeSeriesDataPointDTO     `json:"participants_trend,omitempty"` // 参与人数趋势图数据
	TransactionsTrend []TimeSeriesDataPointDTO     `json:"transactions_trend,omitempty"` // 交易笔数趋势图数据
	PoolTabs          []PoolInfoTabDTO             `json:"pool_tabs,omitempty"`         // 用于切换的奖池Tab信息 (如果有多个Pool)
}

// PoolInfoTabDTO 奖池Tab信息 (用于数据分析前端切换)
type PoolInfoTabDTO struct {
	PoolID   string `json:"pool_id"`   // 奖池ID
	PoolName string `json:"pool_name"` // 奖池显示名称
}

// LeaderboardEntryAdminDTO 排行榜条目 (B端视角)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - Leaderboard 数据"
type LeaderboardEntryAdminDTO struct {
	Rank                int    `json:"rank"`                   // 排名
	UserWalletAddress   string `json:"user_wallet_address"`    // 用户钱包地址
	UserDisplayAddress  string `json:"user_display_address"`   // 用户显示地址 (例如 0x123...abc 或 ENS)
	TradingVolumeUSDStr string `json:"trading_volume_usd_str"` // 交易量 (USD, 格式化)
	TransactionCount    int    `json:"transaction_count"`      // 交易笔数
	// Potentially User profile link / tags if available
}

// RawTransactionListItemDTO 交易明细列表项 (B端视角)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - 明细交易数据"
type RawTransactionListItemDTO struct {
	UserWalletAddress string  `json:"user_wallet_address"` // 用户钱包地址
	NetworkID         string  `json:"network_id"`          // 链ID
	DexID             *string `json:"dex_id,omitempty"`    // DEX ID
	FromTokenSymbol   string  `json:"from_token_symbol"`   // 源代币符号
	FromTokenAmount   string  `json:"from_token_amount"`   // 源代币数量 (格式化)
	ToTokenSymbol     string  `json:"to_token_symbol"`     // 目标代币符号
	ToTokenAmount     string  `json:"to_token_amount"`     // 目标代币数量 (格式化)
	TransactionTime   string  `json:"transaction_time"`    // 交易时间 (ISO8601 UTC)
	TransactionHash   string  `json:"transaction_hash"`    // 交易哈希
	TransactionValueUSDStr *string `json:"transaction_value_usd_str,omitempty"` // 交易USD价值 (可选, 格式化)
	AffectedPoolNames []string `json:"affected_pool_names,omitempty"` // 此交易计入的奖池名称列表
}

### 3.2 JSON-RPC Handler Signatures (B-端管理)

```go
package handler // 假设在 api/jsonrpc/handlers/admin/ 或类似路径

import (
	"context"
	"encoding/json" // For json.RawMessage if complex untyped results are needed

	"taskon/common" // 假设 common.Page, common.ListResponse, common.BaseSuccessResponse 在此
	"taskon/common/jsonrpc" // 假设 jsonrpc 框架的 Request/Response 在此
	"taskon/internal/application/dto" // 假设上面定义的B端DTO在此
	// "taskon/internal/service/admin" // 假设B端管理服务层接口在此
)

// TradingCompetitionAdminHandler 封装了交易大赛B端管理相关的JSON-RPC方法
type TradingCompetitionAdminHandler struct {
	// tradingCompAdminSvc admin.TradingCompetitionAdminService // 实际项目中会注入相应的服务层接口
}

// NewTradingCompetitionAdminHandler 创建 TradingCompetitionAdminHandler 实例
// func NewTradingCompetitionAdminHandler(svc admin.TradingCompetitionAdminService) *TradingCompetitionAdminHandler {
// 	 return &TradingCompetitionAdminHandler{tradingCompAdminSvc: svc}
// }

// CreateTradingCompetition 创建一个新的交易大赛活动。
// Method: tradingCompetitionAdmin_createTradingCompetition
// Params: dto.CreateTradingCompetitionParamsDTO
// Response: dto.TradingCompetitionDetailAdminDTO (返回创建后的完整活动详情)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 创建活动"
func (h *TradingCompetitionAdminHandler) CreateTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailAdminDTO) error {
	var params dto.CreateTradingCompetitionParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err // 框架处理JSON-RPC错误
	}
	// creatorUserID 从 ctx 中获取 (例如通过JWT解析)
	// *result, err = h.tradingCompAdminSvc.CreateTradingCompetition(ctx, creatorUserID, params)
	// return err
	return nil // Placeholder
}

// UpdateTradingCompetition 更新一个已存在的交易大赛活动。
// Method: tradingCompetitionAdmin_updateTradingCompetition
// Params: dto.UpdateTradingCompetitionParamsDTO (其中包含 competition_id)
// Response: dto.TradingCompetitionDetailAdminDTO (返回更新后的完整活动详情)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 编辑活动"
func (h *TradingCompetitionAdminHandler) UpdateTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailAdminDTO) error {
	var params dto.UpdateTradingCompetitionParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// *result, err = h.tradingCompAdminSvc.UpdateTradingCompetition(ctx, creatorUserID, params)
	// return err
	return nil // Placeholder
}

// ListTradingCompetitions 列出B端用户创建的交易大赛活动。
// Method: tradingCompetitionAdmin_listTradingCompetitions
// Params: { "pagination": common.Page, "filter": { "status": "string,optional", "name_contains": "string,optional" } }
// Response: common.ListResponse (of dto.TradingCompetitionListItemDTO)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 查看活动列表"
func (h *TradingCompetitionAdminHandler) ListTradingCompetitions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
		Filter     struct {
			Status       *string `json:"status,omitempty"`
			NameContains *string `json:"name_contains,omitempty"`
		} `json:"filter,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// 允许 filter 和 pagination 为空或部分为空，进行适当处理
	}
	// creatorUserID 从 ctx 中获取
	// items, total, err := h.tradingCompAdminSvc.ListTradingCompetitions(ctx, creatorUserID, params.Pagination, params.Filter)
	// if err != nil { return err }
	// result.Items = items // 需要进行类型断言或使用泛型ListResponse
	// result.Total = total
	return nil // Placeholder
}

// GetTradingCompetitionDetailAdmin 获取特定交易大赛的完整配置详情 (B端管理视角)。
// Method: tradingCompetitionAdmin_getTradingCompetitionDetailAdmin
// Params: { "competition_id": "string" }
// Response: dto.TradingCompetitionDetailAdminDTO
// Requirement Source: Implied by Edit and View operations
func (h *TradingCompetitionAdminHandler) GetTradingCompetitionDetailAdmin(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailAdminDTO) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// *result, err = h.tradingCompAdminSvc.GetTradingCompetitionDetailAdmin(ctx, creatorUserID, params.CompetitionID)
	// return err
	return nil // Placeholder
}

// DeleteTradingCompetition 删除一个交易大赛活动 (仅限特定状态，如DRAFT, UPCOMING)。
// Method: tradingCompetitionAdmin_deleteTradingCompetition
// Params: { "competition_id": "string" }
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 删除活动"
func (h *TradingCompetitionAdminHandler) DeleteTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// err := h.tradingCompAdminSvc.DeleteTradingCompetition(ctx, creatorUserID, params.CompetitionID)
	// if err != nil { result.Success = false; result.Message = err.Error(); return nil }
	// result.Success = true
	return nil // Placeholder
}

// PublishTradingCompetition 发布一个交易大赛活动。
// Method: tradingCompetitionAdmin_publishTradingCompetition
// Params: { "competition_id": "string" }
// Response: dto.TradingCompetitionDetailAdminDTO (返回更新状态后的活动详情)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "三、功能设计 -> 1. 功能地图 -> 发布活动"
func (h *TradingCompetitionAdminHandler) PublishTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailAdminDTO) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// *result, err = h.tradingCompAdminSvc.PublishTradingCompetition(ctx, creatorUserID, params.CompetitionID)
	// return err
	return nil // Placeholder
}

// FetchProjectInfoByTwitter 通过Twitter Handle获取项目的基础信息。
// Method: tradingCompetitionAdmin_fetchProjectInfoByTwitter
// Params: dto.FetchProjectInfoByTwitterParamsDTO
// Response: dto.FetchedProjectInfoDTO
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"
func (h *TradingCompetitionAdminHandler) FetchProjectInfoByTwitter(ctx context.Context, r *jsonrpc.Request, result *dto.FetchedProjectInfoDTO) error {
	var params dto.FetchProjectInfoByTwitterParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompAdminSvc.FetchProjectInfoByTwitter(ctx, params.TwitterHandle)
	// return err
	return nil // Placeholder
}

// CheckProjectTokenLaunchStatus 检查指定项目是否已发行代币。
// Method: tradingCompetitionAdmin_checkProjectTokenLaunchStatus
// Params: dto.CheckProjectTokenParamsDTO
// Response: dto.ProjectTokenCheckResultDTO
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"
func (h *TradingCompetitionAdminHandler) CheckProjectTokenLaunchStatus(ctx context.Context, r *jsonrpc.Request, result *dto.ProjectTokenCheckResultDTO) error {
	var params dto.CheckProjectTokenParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompAdminSvc.CheckProjectTokenLaunchStatus(ctx, params)
	// return err
	return nil // Placeholder
}

// GetCompetitionBudgetStatus 获取活动的预算和充值状态。
// Method: tradingCompetitionAdmin_getCompetitionBudgetStatus
// Params: { "competition_id": "string" }
// Response: dto.CompetitionBudgetStatusDTO
// Requirement Source: Implied by Token充值 and Publish校验
func (h *TradingCompetitionAdminHandler) GetCompetitionBudgetStatus(ctx context.Context, r *jsonrpc.Request, result *dto.CompetitionBudgetStatusDTO) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// *result, err = h.tradingCompAdminSvc.GetCompetitionBudgetStatus(ctx, creatorUserID, params.CompetitionID)
	// return err
	return nil // Placeholder
}

// RecordBudgetDeposit B端记录一笔预算充值 (后台需验证TxHash的有效性)。
// Method: tradingCompetitionAdmin_recordBudgetDeposit
// Params: dto.DepositTokenParamsDTO
// Response: dto.CompetitionBudgetStatusDTO (返回更新后的预算状态)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：Token充值"
func (h *TradingCompetitionAdminHandler) RecordBudgetDeposit(ctx context.Context, r *jsonrpc.Request, result *dto.CompetitionBudgetStatusDTO) error {
	var params dto.DepositTokenParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// depositorUserID 从 ctx 中获取
	// *result, err = h.tradingCompAdminSvc.RecordBudgetDeposit(ctx, depositorUserID, params)
	// return err
	return nil // Placeholder
}

// ---- 数据分析相关API ----

// GetCompetitionDataSummary 获取活动/奖池的汇总数据和趋势图。
// Method: tradingCompetitionAdmin_getCompetitionDataSummary
// Params: dto.CompetitionDataRequestParamsDTO (包含 competition_id, 可选 pool_id, 可选 date_range)
// Response: dto.CompetitionDataSummaryDTO
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - 汇总数据"
func (h *TradingCompetitionAdminHandler) GetCompetitionDataSummary(ctx context.Context, r *jsonrpc.Request, result *dto.CompetitionDataSummaryDTO) error {
	var params dto.CompetitionDataRequestParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// 允许 pool_id 和 date_range 为空
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// *result, err = h.tradingCompAdminSvc.GetCompetitionDataSummary(ctx, creatorUserID, params)
	// return err
	return nil // Placeholder
}

// GetCompetitionLeaderboard 获取活动/奖池的排行榜数据。
// Method: tradingCompetitionAdmin_getCompetitionLeaderboard
// Params: dto.CompetitionDataRequestParamsDTO (包含 competition_id, 可选 pool_id, pagination)
// Response: common.ListResponse (of dto.LeaderboardEntryAdminDTO)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - Leaderboard 数据"
func (h *TradingCompetitionAdminHandler) GetCompetitionLeaderboard(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params dto.CompetitionDataRequestParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// 允许 pool_id 为空
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// items, total, err := h.tradingCompAdminSvc.GetCompetitionLeaderboard(ctx, creatorUserID, params)
	// if err != nil { return err }
	// result.Items = items
	// result.Total = total
	return nil // Placeholder
}

// GetCompetitionRawTransactions 获取活动/奖池的原始交易明细数据。
// Method: tradingCompetitionAdmin_getCompetitionRawTransactions
// Params: dto.CompetitionDataRequestParamsDTO (包含 competition_id, 可选 pool_id, pagination, 可选 date_range/filters)
// Response: common.ListResponse (of dto.RawTransactionListItemDTO)
// Requirement Source: 新需求/整理后的md/new-交易大赛-B端.md - "功能点：数据分析 - 明细交易数据"
func (h *TradingCompetitionAdminHandler) GetCompetitionRawTransactions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params dto.CompetitionDataRequestParamsDTO // อาจจะต้องมี filter เพิ่มเติมสำหรับ tx
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// 允许 pool_id 和 date_range 为空
	}
	// creatorUserID 从 ctx 中获取并校验权限
	// items, total, err := h.tradingCompAdminSvc.GetCompetitionRawTransactions(ctx, creatorUserID, params)
	// if err != nil { return err }
	// result.Items = items
	// result.Total = total
	return nil // Placeholder
}

```

## 4. API 设计 (HTTP - B端管理前端视角)

本部分描述了B端项目方管理后台前端通过HTTP调用的API。为了与现有 `trading_competition_c_end_design.md` 中C端API风格保持一致，所有API均推荐采用POST方法，并将所有参数置于JSON请求体中。这种方式简化了前后端交互，尤其在参数结构复杂或涉及嵌套对象时。

**通用约定:**
-   **Base URL**: `/api/v1/admin/trading-competitions` (示例，具体路径根据项目路由配置)
-   **HTTP Method**: `POST` (所有接口)
-   **认证**: 所有接口均需要B端管理员级别的JWT或会话认证，通过 `Authorization: Bearer <ADMIN_JWT_TOKEN>` 请求头传递。
-   **Content-Type**: 请求和响应体均为 `application/json`。
-   **错误处理**: 遵循项目统一的HTTP错误码和错误响应结构。
-   **请求体**: 所有参数均包含在JSON请求体中。
-   **响应体**: 成功时返回 `200 OK` (或 `201 Created` 用于创建操作)，响应内容为相应的DTO或通用成功响应结构。

### 4.1 活动管理 (CRUD & 状态变更)

1.  **创建交易大赛活动**
    *   **Endpoint**: `/create`
    *   **Request Body**: `dto.CreateTradingCompetitionParamsDTO`
    *   **Response Body**: `dto.TradingCompetitionDetailAdminDTO` (包含新创建活动的完整详情和ID)
    *   **描述**: 项目方创建一个全新的交易大赛活动，包括所有基础配置、项目信息、奖池和页面样式。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "三、功能设计 -> 1. 功能地图 -> 创建活动"

2.  **更新交易大赛活动**
    *   **Endpoint**: `/update`
    *   **Request Body**: `dto.UpdateTradingCompetitionParamsDTO` (必须包含 `competition_id`)
    *   **Response Body**: `dto.TradingCompetitionDetailAdminDTO` (包含更新后活动的完整详情)
    *   **描述**: 项目方修改已存在的交易大赛活动的各项配置。活动状态会影响可编辑字段。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "三、功能设计 -> 1. 功能地图 -> 编辑活动"

3.  **获取交易大赛活动列表 (B端管理视角)**
    *   **Endpoint**: `/list`
    *   **Request Body**:
        ```json
        {
          "pagination": { // dto.common.Page
            "page_no": "int, optional, default: 1",
            "page_size": "int, optional, default: 10"
          },
          "filter": {
            "status": "string, optional", // 活动状态 (DRAFT, UPCOMING, ONGOING, ENDED, ...)
            "name_contains": "string, optional" // 活动名称模糊查询
          }
        }
        ```
    *   **Response Body**: `common.ListResponse<dto.TradingCompetitionListItemDTO>`
    *   **描述**: 获取当前项目方创建的交易大赛活动列表，支持分页和筛选。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "三、功能设计 -> 1. 功能地图 -> 查看活动列表"

4.  **获取交易大赛活动详情 (B端管理视角)**
    *   **Endpoint**: `/detail`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 要获取详情的活动ID
        }
        ```
    *   **Response Body**: `dto.TradingCompetitionDetailAdminDTO`
    *   **描述**: 获取指定交易大赛活动的完整配置信息，用于编辑页面展示或查看。
    *   **Requirement Source**: Implied by Edit and View operations, covering all config sections.

5.  **删除交易大赛活动**
    *   **Endpoint**: `/delete`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 要删除的活动ID
        }
        ```
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **描述**: 删除一个交易大赛活动。通常只允许在活动为 `DRAFT` 或 `UPCOMING` 状态时操作，并进行二次确认。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "三、功能设计 -> 1. 功能地图 -> 删除活动"

6.  **发布交易大赛活动**
    *   **Endpoint**: `/publish`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 要发布的活动ID
        }
        ```
    *   **Response Body**: `dto.TradingCompetitionDetailAdminDTO` (返回更新状态后的活动详情)
    *   **描述**: 发布一个已配置完成且预算校验通过的交易大赛活动。活动状态将变为 `UPCOMING` 或 `ONGOING`。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "三、功能设计 -> 1. 功能地图 -> 发布活动"

### 4.2 辅助配置接口

Base URL for utils: `/api/v1/admin/utils` (示例)

1.  **通过Twitter Handle获取项目信息**
    *   **Endpoint**: `/fetch-project-info-by-twitter` (under utils base path)
    *   **Request Body**: `dto.FetchProjectInfoByTwitterParamsDTO`
    *   **Response Body**: `dto.FetchedProjectInfoDTO`
    *   **描述**: 根据提供的Twitter Handle，尝试从外部服务（如Twitter API）抓取项目名称和Logo等信息，辅助B端快速配置参与项目。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"

2.  **检查项目是否已发币**
    *   **Endpoint**: `/check-project-token-launch` (under utils base path)
    *   **Request Body**: `dto.CheckProjectTokenParamsDTO`
    *   **Response Body**: `dto.ProjectTokenCheckResultDTO`
    *   **描述**: 检查指定的项目（根据名称或Twitter Handle）是否已在主流平台（如CMC/Alpha库）发行代币，并返回检查结果及相关代币信息（如果已发币）。用于引导项目方设置专属奖池。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "二、业务流程 -> 2. 关键子流程图 -> 项目信息配置流程"

### 4.3 预算与充值管理

Base URL: `/api/v1/admin/trading-competitions` (continued)

1.  **获取活动预算状态**
    *   **Endpoint**: `/budget-status`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 活动ID
        }
        ```
    *   **Response Body**: `dto.CompetitionBudgetStatusDTO`
    *   **描述**: 查询指定交易大赛当前的预算需求、已充值金额及是否充足等状态信息。
    *   **Requirement Source**: Implied by "功能点：Token充值" and Publish校验

2.  **记录预算充值**
    *   **Endpoint**: `/record-deposit`
    *   **Request Body**: `dto.DepositTokenParamsDTO` (包含 `competition_id`, 代币信息, 数量, 充值网络及TxHash)
    *   **Response Body**: `dto.CompetitionBudgetStatusDTO` (返回更新后的预算状态)
    *   **描述**: B端项目方在链上完成充值后，通过此接口向系统记录该笔充值。后端需对交易哈希进行验证。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "功能点：Token充值"

### 4.4 数据分析接口

Base URL: `/api/v1/admin/trading-competitions/data` (示例)

1.  **获取活动数据汇总与趋势**
    *   **Endpoint**: `/summary`
    *   **Request Body**: `dto.CompetitionDataRequestParamsDTO` (包含 `competition_id`, 可选 `pool_id`, 可选 `date_range_start`, `date_range_end`)
    *   **Response Body**: `dto.CompetitionDataSummaryDTO`
    *   **描述**: 获取指定活动（或其中特定奖池）的汇总数据（总交易量、参与人数、交易笔数及其日增）和各项指标的时间趋势图数据。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "功能点：数据分析 - 汇总数据"

2.  **获取活动排行榜**
    *   **Endpoint**: `/leaderboard`
    *   **Request Body**: `dto.CompetitionDataRequestParamsDTO` (包含 `competition_id`, 可选 `pool_id`, `pagination`)
    *   **Response Body**: `common.ListResponse<dto.LeaderboardEntryAdminDTO>`
    *   **描述**: 获取指定活动（或其中特定奖池）的用户交易量排行榜。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "功能点：数据分析 - Leaderboard 数据"

3.  **获取活动原始交易明细**
    *   **Endpoint**: `/transactions`
    *   **Request Body**: `dto.CompetitionDataRequestParamsDTO` (包含 `competition_id`, 可选 `pool_id`, `pagination`, 可选日期范围或其它筛选条件)
    *   **Response Body**: `common.ListResponse<dto.RawTransactionListItemDTO>`
    *   **描述**: 获取指定活动（或其中特定奖池）的符合条件的原始交易明细列表，支持分页和筛选。
    *   **Requirement Source**: `新需求/整理后的md/new-交易大赛-B端.md` - "功能点：数据分析 - 明细交易数据"

---
**备注:**

- 上述HTTP API设计为B端管理后台提供了全面的接口支持。实际部署时，Base URL和具体endpoint路径可根据项目整体API网关和路由策略进行调整。
- 对于列表查询接口，除了基础的分页，还可以根据实际需求在请求DTO中添加更丰富的筛选和排序参数。
- 错误处理和具体的认证/授权逻辑需要在服务层实现，并遵循项目统一规范。
- 部分复杂操作（如全量更新活动配置中的多个奖池）可能需要考虑更细粒度的API或在服务层处理复杂的更新逻辑，以确保原子性和数据一致性。