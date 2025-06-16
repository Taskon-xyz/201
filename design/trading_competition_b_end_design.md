# 交易大赛 - B端功能设计文档

## 1. 功能概述

本文档根据 `新需求/整理后的md/new-交易大赛-B端.md` 设计交易大赛B端（项目方）功能。项目方可以创建和管理自定义的链上交易竞赛活动，配置活动信息、项目信息、奖励规则和页面样式。活动结束后，项目方可以查看详细的数据分析，包括参与人数、交易量、排行榜等。

## 2. 数据库设计 (MySQL)

参考 `taskon-actions/dbdesign.md` 中的设计规范。

```sql
-- trading_competitions: 存储交易大赛活动的基本信息
CREATE TABLE `trading_competitions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `name` VARCHAR(255) NOT NULL COMMENT '活动名称',
  `creator_user_id` BIGINT UNSIGNED NOT NULL COMMENT '创建活动的用户ID (关联到 taskon-server 的用户表)',
  `creator_entity_id` BIGINT UNSIGNED NOT NULL COMMENT '创建活动的实体ID (如社区ID, 关联到 taskon-server 的社区/项目表)',
  `start_time` DATETIME(3) NOT NULL COMMENT '活动开始时间',
  `end_time` DATETIME(3) NOT NULL COMMENT '活动结束时间',
  `banner_pc_url` VARCHAR(512) DEFAULT NULL COMMENT 'PC端Banner图片URL',
  `banner_mobile_url` VARCHAR(512) DEFAULT NULL COMMENT 'Mobile端Banner图片URL',
  `dot_animation_enabled` BOOLEAN DEFAULT FALSE COMMENT '是否启用点状动画',
  `page_style_config` JSON DEFAULT NULL COMMENT '页面样式配置 (例如: {\"background_color\": \"#FFFFFF\", \"theme_color\": \"#0000FF\"})',
  `network_id` VARCHAR(100) NOT NULL COMMENT '交易活动所在的链ID (例如: "1" for Ethereum Mainnet, "56" for BNB Chain)',
  `dex_id` VARCHAR(100) NOT NULL COMMENT '交易活动指定的DEX ID (例如: "taskon_aggregator", "pancakeswap_v2")',
  `status` VARCHAR(20) NOT NULL COMMENT '活动状态 (DRAFT, UPCOMING, ONGOING, ENDED, ARCHIVED)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  `deleted_at` DATETIME(3) DEFAULT NULL COMMENT '软删除时间，NULL表示未删除',
  PRIMARY KEY (`id`),
  INDEX `idx_creator_entity_id_status` (`creator_entity_id`, `status` COMMENT '根据创建实体和状态查询活动'),
  INDEX `idx_status_start_end_time` (`status`, `start_time`, `end_time` COMMENT '根据状态和时间范围查询活动')
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛活动主表，存储活动的核心配置信息';

-- trading_competition_projects: 存储参与交易大赛的项目方信息
CREATE TABLE `trading_competition_projects` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `project_name` VARCHAR(255) NOT NULL COMMENT '项目名称',
  `project_logo_url` VARCHAR(512) DEFAULT NULL COMMENT '项目Logo URL',
  `project_twitter_handle` VARCHAR(100) DEFAULT NULL COMMENT '项目Twitter Handle',
  `display_order` INT NOT NULL DEFAULT 0 COMMENT '显示顺序，数字越小越靠前',
  `is_creator_project` BOOLEAN DEFAULT FALSE COMMENT '标记是否为活动创建方自身的项目',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_competition_id_display_order` (`competition_id`, `display_order` COMMENT '根据活动ID和显示顺序查询'),
  CONSTRAINT `fk_tcp_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛参与项目表，存储活动中展示的各个项目信息';

-- trading_competition_pools: 存储交易大赛的奖池配置
CREATE TABLE `trading_competition_pools` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `pool_type` VARCHAR(20) NOT NULL COMMENT '奖池类型 (BASIC: 基础奖池, SPECIFIED: 指定代币奖池)',
  `name` VARCHAR(255) NOT NULL COMMENT '奖池名称 (例如: \"Basic Pool\", \"ENA Pool\")',
  `is_enabled` BOOLEAN DEFAULT TRUE COMMENT '是否启用此奖池 (主要用于多项目方时，基础奖池的可选开关)',
  `reward_token_address` VARCHAR(255) NOT NULL COMMENT '奖励代币的合约地址',
  `reward_token_symbol` VARCHAR(50) NOT NULL COMMENT '奖励代币的符号 (例如: USDT, ETH)',
  `reward_token_decimals` INT NOT NULL COMMENT '奖励代币的精度 (例如: 6 for USDT, 18 for ETH)',
  `total_reward_amount_raw` VARCHAR(78) NOT NULL COMMENT '奖励代币总数量 (以原始精度的字符串形式存储，例如 \"1000000000\" 表示1000个6位精度的USDT)',
  `config_json` JSON NOT NULL COMMENT '奖池特定配置 (JSON格式，存储交易对等详细规则)',
  `winner_eligibility_type` VARCHAR(50) DEFAULT NULL COMMENT '获奖资格类型 (VOLUME_THRESHOLD: 交易量门槛, TOP_N_RANK: 排行榜前N名)',
  `winner_eligibility_value_usd_raw` VARCHAR(78) DEFAULT NULL COMMENT '获奖资格值 - 交易量门槛 (USD计价，原始精度的字符串形式存储)',
  `winner_eligibility_value_rank` INT DEFAULT NULL COMMENT '获奖资格值 - 排行榜名次 (例如: 10 表示前10名)',
  `display_order` INT NOT NULL DEFAULT 0 COMMENT '显示顺序 (主要用于指定奖池，数字越小越靠前)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_competition_id_pool_type_display_order` (`competition_id`, `pool_type`, `display_order` COMMENT '根据活动ID、奖池类型和顺序查询'),
  CONSTRAINT `fk_tcs_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛奖池配置表，定义每个奖池的奖励规则和交易要求';

/*
config_json 结构示例:
Basic Pool:
{
  "swap_from_type": "ANY" | "SPECIFIC",
  "swap_from_tokens": ["0x...", "0x..."], // (可选) Token地址列表
  "swap_to_type": "ANY" | "SPECIFIC",
  "swap_to_tokens": ["0x...", "0x..."],   // (可选) Token地址列表
  "exclude_same_underlying_trades": true | false // (可选) 当swap_from/to均为ANY时
}

Specified Pool:
{
  "swap_from_type": "ANY", // 固定为ANY
  // "swap_to_tokens" 隐式为奖池的 reward_token_address
}
*/

-- trading_competition_deposits: 存储项目方为交易大赛充值的代币预算
CREATE TABLE `trading_competition_deposits` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `token_address` VARCHAR(255) NOT NULL COMMENT '充值代币的合约地址',
  `token_symbol` VARCHAR(50) NOT NULL COMMENT '充值代币的符号',
  `token_decimals` INT NOT NULL COMMENT '充值代币的精度',
  `deposited_amount_raw` VARCHAR(78) NOT NULL DEFAULT '0' COMMENT '已充值数量 (以原始精度的字符串形式存储)',
  `last_deposit_time` DATETIME(3) DEFAULT NULL COMMENT '最后一次充值时间',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_competition_token` (`competition_id`, `token_address` COMMENT '确保同一活动同种代币的充值记录唯一'),
  CONSTRAINT `fk_tcd_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛项目方充值预算表，记录各代币的充值情况';

-- trading_competition_user_volume: 存储用户在各奖池中的交易统计数据 (由后端数据处理服务更新)
CREATE TABLE `trading_competition_user_volume` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `pool_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的奖池ID (外键, trading_competition_pools.id)',
  `user_wallet_address` VARCHAR(255) NOT NULL COMMENT '用户钱包地址',
  `total_trading_volume_usd_raw` VARCHAR(78) NOT NULL DEFAULT '0' COMMENT '累计有效交易额 (USD计价, 原始精度的字符串形式存储)',
  `total_transactions` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '累计有效交易笔数',
  `last_processed_tx_time` DATETIME(3) DEFAULT NULL COMMENT '该用户此奖池数据最后处理的交易时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_pool_user` (`pool_id`, `user_wallet_address` COMMENT '确保用户在同一奖池的统计记录唯一'),
  INDEX `idx_comp_pool_vol` (`competition_id`, `pool_id`, `total_trading_volume_usd_raw`(10) COMMENT '用于排行榜按交易量排序查询'),
  CONSTRAINT `fk_tcuv_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `fk_tcuv_pool_id` FOREIGN KEY (`pool_id`) REFERENCES `trading_competition_pools` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛用户交易量统计表，用于排行榜和数据分析';

-- trading_competition_raw_transactions: 存储符合活动基本条件的原始交易数据 (可选，但推荐用于B端导出和审计)
CREATE TABLE `trading_competition_raw_transactions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `user_wallet_address` VARCHAR(255) NOT NULL COMMENT '用户钱包地址',
  `tx_hash` VARCHAR(255) NOT NULL COMMENT '交易哈希',
  `network_id` VARCHAR(100) NOT NULL COMMENT '交易发生的链ID',
  `from_token_address` VARCHAR(255) NOT NULL COMMENT 'Swap出的代币地址',
  `from_token_amount_raw` VARCHAR(78) NOT NULL COMMENT 'Swap出的代币数量 (原始精度字符串)',
  `to_token_address` VARCHAR(255) NOT NULL COMMENT 'Swap入的代币地址',
  `to_token_amount_raw` VARCHAR(78) NOT NULL COMMENT 'Swap入的代币数量 (原始精度字符串)',
  `transaction_time` DATETIME(3) NOT NULL COMMENT '交易发生时间',
  `volume_usd_raw` VARCHAR(78) NOT NULL COMMENT '该笔交易的USD计价交易额 (原始精度字符串)',
  `contributed_pools_info` JSON DEFAULT NULL COMMENT '记录此交易对哪些奖池有贡献及其有效性 (例如: [{\"pool_id\": 123, \"is_valid_for_pool\": true, \"contributed_volume_usd_raw\": \"10050\"}] )',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录入库时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_competition_tx_hash` (`competition_id`, `tx_hash` COMMENT '确保同一活动内的交易哈希唯一'),
  INDEX `idx_comp_user_time` (`competition_id`, `user_wallet_address`, `transaction_time` COMMENT '按活动、用户和时间查询交易'),
  -- INDEX `idx_competition_pool_id_in_json` (如果DB支持JSON路径索引，可考虑添加，用于按奖池筛选明细)
  CONSTRAINT `fk_tcrt_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛原始交易记录表，用于数据明细导出和审计';
```

## 3. API 设计 (JSON-RPC)

遵循 `taskon-actions/guideline.md`。

**DTOs (B-端)**
Package `dto` (位于 `internal/application/dto/` 或类似路径)

```go
package dto

import (
	"encoding/json"
	"taskon/common" // 假设 common.Page 和 common.ListResponse 在此
)

// TradingCompetitionProjectParamsDTO 定义了创建或更新活动时，项目信息的参数
type TradingCompetitionProjectParamsDTO struct {
	ProjectName          string `json:"project_name"`                      // 项目名称
	ProjectLogoURL       string `json:"project_logo_url,omitempty"`        // 项目Logo URL (可选)
	ProjectTwitterHandle string `json:"project_twitter_handle,omitempty"`  // 项目Twitter Handle (可选)
	DisplayOrder         *int   `json:"display_order,omitempty"`           // 显示顺序 (可选, 服务端可管理默认项目顺序)
}

// TradingCompetitionPoolConfigDTO 定义了奖池中交易对相关的配置 (用于 trading_competition_pools.config_json)
type TradingCompetitionPoolConfigDTO struct {
	SwapFromType                string   `json:"swap_from_type"`                             // 交易源类型: "ANY" 或 "SPECIFIC"
	SwapFromTokens              []string `json:"swap_from_tokens,omitempty"`                 // 交易源Token地址列表 (当 SwapFromType 为 "SPECIFIC" 时)
	SwapToType                  string   `json:"swap_to_type"`                               // 交易目标类型: "ANY" 或 "SPECIFIC"
	SwapToTokens                []string `json:"swap_to_tokens,omitempty"`                   // 交易目标Token地址列表 (当 SwapToType 为 "SPECIFIC" 时)
	ExcludeSameUnderlyingTrades *bool    `json:"exclude_same_underlying_trades,omitempty"` // 是否排除同底层资产交易 (例如 USDT/USDC, ETH/WETH; 当 SwapFromType 和 SwapToType 均为 "ANY" 时)
}

// TradingCompetitionPoolParamsDTO 定义了创建或更新活动时，奖池信息的参数
type TradingCompetitionPoolParamsDTO struct {
	PoolType                     string                           `json:"pool_type"`                                    // 奖池类型: "BASIC" 或 "SPECIFIED"
	Name                         string                           `json:"name"`                                         // 奖池名称
	IsEnabled                    *bool                            `json:"is_enabled,omitempty"`                         // 是否启用此奖池 (主要用于多项目方时的Basic Pool, 可选)
	RewardTokenAddress           string                           `json:"reward_token_address"`                         // 奖励代币的合约地址
	RewardTokenSymbol            string                           `json:"reward_token_symbol"`                          // 奖励代币的符号
	RewardTokenDecimals          int                              `json:"reward_token_decimals"`                        // 奖励代币的精度
	TotalRewardAmountStr         string                           `json:"total_reward_amount_str"`                        // 奖励代币总数量 (使用字符串以支持大数)
	Config                       TradingCompetitionPoolConfigDTO  `json:"config"`                                       // 奖池特定配置 (交易对等)
	WinnerEligibilityType        *string                          `json:"winner_eligibility_type,omitempty"`            // 获奖资格类型: "VOLUME_THRESHOLD" (交易量门槛), "TOP_N_RANK" (排行榜前N名), 可选
	WinnerEligibilityValueUSDStr *string                          `json:"winner_eligibility_value_usd_str,omitempty"`   // 获奖资格值 - 交易量门槛 (USD计价, 使用字符串以支持大数, 可选)
	WinnerEligibilityValueRank   *int                             `json:"winner_eligibility_value_rank,omitempty"`      // 获奖资格值 - 排行榜名次 (例如: 10 表示前10名, 可选)
	DisplayOrder                 *int                             `json:"display_order,omitempty"`                        // 显示顺序 (可选)
}

// TradingCompetitionPageStyleDTO 定义了活动页面的样式配置
type TradingCompetitionPageStyleDTO struct {
	BackgroundColor    string `json:"background_color,omitempty"`     // 背景色 (例如: "#FFFFFF", 可选)
	BackgroundImageURL string `json:"background_image_url,omitempty"` // 背景图片URL (可选)
	ThemeColor         string `json:"theme_color,omitempty"`          // 主题色 (例如: "#0000FF", 可选)
}

// CreateTradingCompetitionParamsDTO 定义了创建交易大赛的参数
type CreateTradingCompetitionParamsDTO struct {
	Name                 string                               `json:"name"`                                   // 活动名称
	StartTime            string                               `json:"start_time"`                             // 活动开始时间 (ISO8601 UTC 字符串格式, 例如: "2023-10-26T10:00:00Z")
	EndTime              string                               `json:"end_time"`                               // 活动结束时间 (ISO8601 UTC 字符串格式)
	BannerPCURL          string                               `json:"banner_pc_url,omitempty"`                // PC端Banner图片URL (可选)
	BannerMobileURL      string                               `json:"banner_mobile_url,omitempty"`            // Mobile端Banner图片URL (可选)
	DotAnimationEnabled  *bool                                `json:"dot_animation_enabled,omitempty"`        // 是否启用点状动画 (可选, 默认为 false)
	PageStyleConfig      *TradingCompetitionPageStyleDTO      `json:"page_style_config,omitempty"`            // 页面样式配置 (可选)
	NetworkID            string                               `json:"network_id"`                             // 交易活动所在的链ID
	DexID                string                               `json:"dex_id"`                                 // 交易活动指定的DEX ID
	Projects             []TradingCompetitionProjectParamsDTO `json:"projects"`                               // 参与项目列表 (至少一个，创建者项目信息由后端根据调用者身份自动填充或客户端明确传入)
	Pools                []TradingCompetitionPoolParamsDTO    `json:"pools"`                                  // 奖池列表 (至少一个)
}

// UpdateTradingCompetitionParamsDTO 定义了更新交易大赛的参数
type UpdateTradingCompetitionParamsDTO struct {
	CompetitionID        string                                `json:"competition_id"`                        // 要更新的活动ID (通常为数字字符串)
	Name                 *string                               `json:"name,omitempty"`                        // 活动名称 (可选)
	StartTime            *string                               `json:"start_time,omitempty"`                  // 活动开始时间 (可选, ISO8601 UTC 字符串格式)
	EndTime              *string                               `json:"end_time,omitempty"`                    // 活动结束时间 (可选, ISO8601 UTC 字符串格式)
	BannerPCURL          *string                               `json:"banner_pc_url,omitempty"`               // PC端Banner图片URL (可选)
	BannerMobileURL      *string                               `json:"banner_mobile_url,omitempty"`           // Mobile端Banner图片URL (可选)
	DotAnimationEnabled  *bool                                 `json:"dot_animation_enabled,omitempty"`       // 是否启用点状动画 (可选)
	PageStyleConfig      *TradingCompetitionPageStyleDTO       `json:"page_style_config,omitempty"`           // 页面样式配置 (可选)
	NetworkID            *string                               `json:"network_id,omitempty"`                  // 交易活动所在的链ID (可选, 若活动未开始)
	DexID                *string                               `json:"dex_id,omitempty"`                      // 交易活动指定的DEX ID (可选, 若活动未开始)
	Projects             *[]TradingCompetitionProjectParamsDTO `json:"projects,omitempty"`                    // 参与项目列表 (可选, 若提供则全量更新)
	Pools                *[]TradingCompetitionPoolParamsDTO    `json:"pools,omitempty"`                       // 奖池列表 (可选, 若提供则全量更新)
}

// TradingCompetitionListItemDTO 定义了活动列表项的结构
type TradingCompetitionListItemDTO struct {
	ID                     string `json:"id"`                       // 活动ID (数字字符串)
	Name                   string `json:"name"`                     // 活动名称
	StartTime              string `json:"start_time"`               // 活动开始时间 (ISO8601 UTC 字符串)
	EndTime                string `json:"end_time"`                 // 活动结束时间 (ISO8601 UTC 字符串)
	Status                 string `json:"status"`                   // 活动状态 (DRAFT, UPCOMING, ONGOING, ENDED, ARCHIVED)
	NetworkID              string `json:"network_id"`               // 链ID
	DexID                  string `json:"dex_id"`                   // DEX ID
	TotalBudgetSummaryStr  string `json:"total_budget_summary_str"` // 总预算摘要 (例如 "1000 USDT, 500 ENA")
	CreatedAt              string `json:"created_at"`               // 创建时间 (ISO8601 UTC 字符串)
}

// DepositedTokenStatusDTO 定义了单个代币的充值状态，用于预算检查
type DepositedTokenStatusDTO struct {
	TokenAddress         string `json:"token_address"`          // 代币合约地址
	TokenSymbol          string `json:"token_symbol"`           // 代币符号
	TokenDecimals        int    `json:"token_decimals"`         // 代币精度
	DepositedAmountStr   string `json:"deposited_amount_str"`   // 已充值数量 (字符串表示, 原始精度)
	RequiredAmountStr    string `json:"required_amount_str"`    // 活动需求总数量 (字符串表示, 原始精度)
	IsSufficient         bool   `json:"is_sufficient"`          // 当前充值是否满足需求
}

// TradingCompetitionDetailDTO 定义了活动详情的结构，供B端查看和编辑
type TradingCompetitionDetailDTO struct {
	ID                   string                               `json:"id"`                                  // 活动ID (数字字符串)
	Name                 string                               `json:"name"`                                // 活动名称
	CreatorUserID        string                               `json:"creator_user_id"`                     // 创建者用户ID (数字字符串)
	CreatorEntityID      string                               `json:"creator_entity_id"`                   // 创建者实体ID (数字字符串)
	StartTime            string                               `json:"start_time"`                          // 活动开始时间 (ISO8601 UTC 字符串)
	EndTime              string                               `json:"end_time"`                              // 活动结束时间 (ISO8601 UTC 字符串)
	BannerPCURL          string                               `json:"banner_pc_url,omitempty"`                 // PC端Banner图片URL
	BannerMobileURL      string                               `json:"banner_mobile_url,omitempty"`               // Mobile端Banner图片URL
	DotAnimationEnabled  bool                                 `json:"dot_animation_enabled"`                     // 是否启用点状动画
	PageStyleConfig      *TradingCompetitionPageStyleDTO      `json:"page_style_config,omitempty"`                 // 页面样式配置
	NetworkID            string                               `json:"network_id"`                                // 链ID
	DexID                string                               `json:"dex_id"`                                    // DEX ID
	Status               string                               `json:"status"`                                    // 活动状态
	Projects             []TradingCompetitionProjectParamsDTO `json:"projects"`                                    // 项目列表 (这里复用ParamsDTO，展示时可能字段一致)
	Pools                []TradingCompetitionPoolParamsDTO    `json:"pools"`                                         // 奖池列表 (同上)
	DepositedTokens      []DepositedTokenStatusDTO            `json:"deposited_tokens,omitempty"`                    // 各代币充值状态 (发布前检查)
	CreatedAt            string                               `json:"created_at"`                                    // 创建时间 (ISO8601 UTC 字符串)
	UpdatedAt            string                               `json:"updated_at"`                                    // 最后更新时间 (ISO8601 UTC 字符串)
}

// DataAnalysisLeaderboardItemDTO 定义了数据分析排行榜条目
type DataAnalysisLeaderboardItemDTO struct {
	Rank                 int    `json:"rank"`                   // 排名
	UserAddress          string `json:"user_address"`           // 用户钱包地址
	TradingVolumeUSDStr  string `json:"trading_volume_usd_str"` // 交易量 (USD, 字符串表示, 带适当小数位)
	Transactions         int    `json:"transactions"`           // 交易笔数
}

// DataAnalysisTransactionItemDTO 定义了数据分析明细交易条目
type DataAnalysisTransactionItemDTO struct {
	WalletAddress       string `json:"wallet_address"`        // 用户钱包地址
	NetworkID           string `json:"network_id"`            // 交易发生的链ID
	FromTokenSymbol     string `json:"from_token_symbol"`     // Swap出的代币符号
	FromTokenAmountStr  string `json:"from_token_amount_str"` // Swap出的代币数量 (字符串表示, 带适当小数位)
	ToTokenSymbol       string `json:"to_token_symbol"`       // Swap入的代币符号
	ToTokenAmountStr    string `json:"to_token_amount_str"`   // Swap入的代币数量 (字符串表示, 带适当小数位)
	TransactionTime     string `json:"transaction_time"`      // 交易发生时间 (ISO8601 UTC 字符串)
	TxHash              string `json:"tx_hash"`               // 交易哈希
	VolumeUSDStr        string `json:"volume_usd_str"`        // 该笔交易的USD价值 (字符串表示, 带适当小数位)
}

// --- Request/Response DTOs for API Endpoints (Wrapper DTOs if needed by JSON-RPC framework) ---

// BaseSuccessResponse DTO for simple success/failure
type BaseSuccessResponse struct {
	Success bool   `json:"success"`
	Message string `json:"message,omitempty"`
}

// GetProjectTokenInfoByTwitterRequest DTO
type GetProjectTokenInfoByTwitterRequest struct {
	TwitterHandle string `json:"twitter_handle"`
}

// ProjectTokenInfoDTO for GetProjectTokenInfoByTwitterResponse
type ProjectTokenInfoDTO struct {
	Symbol  string `json:"symbol"`
	Address string `json:"address"`
	Name    string `json:"name"`
}

// GetProjectTokenInfoByTwitterResponse DTO
type GetProjectTokenInfoByTwitterResponse struct {
	ProjectName string                `json:"project_name"`
	LogoURL     string                `json:"logo_url,omitempty"`
	Tokens      []ProjectTokenInfoDTO `json:"tokens,omitempty"`
}
```

**API Endpoints (Go Handler Signatures)**
Package `handler` (位于 `api/jsonrpc/handlers/` 或类似路径)

```go
package handler

import (
	"context"
	
	"taskon/common" // 假设 common.Page 和 common.ListResponse 在此
	"taskon/common/jsonrpc" // 假设 jsonrpc 框架的 Request/Response 在此
	"taskon/internal/application/dto" // 假设上面定义的DTO在此
	// "taskon/internal/service" // 假设服务层接口在此
)

// TradingCompetitionAdminHandler 封装了交易大赛B端管理的JSON-RPC方法
type TradingCompetitionAdminHandler struct {
	// tradingCompetitionAdminSvc service.TradingCompetitionAdminService // 实际项目中会注入服务
}

// NewTradingCompetitionAdminHandler 创建 TradingCompetitionAdminHandler 实例
// func NewTradingCompetitionAdminHandler(svc service.TradingCompetitionAdminService) *TradingCompetitionAdminHandler {
//	 return &TradingCompetitionAdminHandler{tradingCompetitionAdminSvc: svc}
// }

// CreateTradingCompetition 创建新的交易大赛
// Method: tradingCompetitionAdmin_createTradingCompetition
func (h *TradingCompetitionAdminHandler) CreateTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailDTO) error {
	var params dto.CreateTradingCompetitionParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err // Or a specific JsonRpcInvalidParams error
	}
	// *result, err = h.tradingCompetitionAdminSvc.CreateTradingCompetition(ctx, params)
	// return err
	return nil // Placeholder
}

// UpdateTradingCompetition 更新已有的交易大赛
// Method: tradingCompetitionAdmin_updateTradingCompetition
func (h *TradingCompetitionAdminHandler) UpdateTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailDTO) error {
	var params dto.UpdateTradingCompetitionParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionAdminSvc.UpdateTradingCompetition(ctx, params)
	// return err
	return nil // Placeholder
}

// SaveTradingCompetitionAsDraft 将交易大赛保存为草稿
// Method: tradingCompetitionAdmin_saveTradingCompetitionAsDraft
func (h *TradingCompetitionAdminHandler) SaveTradingCompetitionAsDraft(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailDTO) error {
	// Params can be CreateTradingCompetitionParamsDTO or UpdateTradingCompetitionParamsDTO
	// Need to determine based on presence of competition_id, or use a wrapper DTO.
	// For simplicity, service layer might handle unmarshalling json.RawMessage.
	var rawParams json.RawMessage
	if err := jsonrpc.UnmarshalParams(r.Params, &rawParams); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionAdminSvc.SaveTradingCompetitionAsDraft(ctx, rawParams)
	// return err
	return nil // Placeholder
}

// ListTradingCompetitions 列出交易大赛
// Method: tradingCompetitionAdmin_listTradingCompetitions
func (h *TradingCompetitionAdminHandler) ListTradingCompetitions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error { // Assuming result is common.ListResponse of TradingCompetitionListItemDTO
	var params struct {
		Filter     *dto.ListTradingCompetitionsRequestFilter `json:"filter,omitempty"`
		Pagination *common.Page                          `json:"pagination,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// listItems, total, err := h.tradingCompetitionAdminSvc.ListTradingCompetitions(ctx, params.Filter, params.Pagination)
	// if err != nil { return err }
	// result.Items = listItems // Needs type assertion or generic handling
	// result.Total = total
	return nil // Placeholder
}

// GetTradingCompetitionDetail 获取交易大赛详情
// Method: tradingCompetitionAdmin_getTradingCompetitionDetail
func (h *TradingCompetitionAdminHandler) GetTradingCompetitionDetail(ctx context.Context, r *jsonrpc.Request, result *dto.TradingCompetitionDetailDTO) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionAdminSvc.GetTradingCompetitionDetail(ctx, params.CompetitionID)
	// return err
	return nil // Placeholder
}

// PublishTradingCompetition 发布交易大赛
// Method: tradingCompetitionAdmin_publishTradingCompetition
func (h *TradingCompetitionAdminHandler) PublishTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.BaseSuccessResponse) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// updatedCompetition, err := h.tradingCompetitionAdminSvc.PublishTradingCompetition(ctx, params.CompetitionID)
	// if err != nil { result.Success = false; result.Message = err.Error(); return nil }
	// result.Success = true
	// // Optionally include updatedCompetition in a more complex response if needed.
	return nil // Placeholder
}

// DeleteTradingCompetition 删除交易大赛
// Method: tradingCompetitionAdmin_deleteTradingCompetition
func (h *TradingCompetitionAdminHandler) DeleteTradingCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.BaseSuccessResponse) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.tradingCompetitionAdminSvc.DeleteTradingCompetition(ctx, params.CompetitionID)
	// if err != nil { result.Success = false; result.Message = err.Error(); return nil }
	// result.Success = true
	return nil // Placeholder
}

// DepositTokenForCompetition 为交易大赛充值代币
// Method: tradingCompetitionAdmin_depositTokenForCompetition
func (h *TradingCompetitionAdminHandler) DepositTokenForCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.BaseSuccessResponse) error { // Response could be more specific
	var params struct {
		CompetitionID    string `json:"competition_id"`
		TokenAddress     string `json:"token_address"`
		DepositAmountStr string `json:"deposit_amount_str"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// updatedDeposit, err := h.tradingCompetitionAdminSvc.DepositTokenForCompetition(ctx, params.CompetitionID, params.TokenAddress, params.DepositAmountStr)
	// if err != nil { result.Success = false; result.Message = err.Error(); return nil }
	// result.Success = true
	// // Populate result with updatedDeposit details if needed in a custom response DTO
	return nil // Placeholder
}

// GetCompetitionBudgetStatus 获取交易大赛预算状态
// Method: tradingCompetitionAdmin_getCompetitionBudgetStatus
func (h *TradingCompetitionAdminHandler) GetCompetitionBudgetStatus(ctx context.Context, r *jsonrpc.Request, result *struct{Deposits []dto.DepositedTokenStatusDTO}) error {
	var params struct {
		CompetitionID string `json:"competition_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// deposits, err := h.tradingCompetitionAdminSvc.GetCompetitionBudgetStatus(ctx, params.CompetitionID)
	// if err != nil { return err }
	// result.Deposits = deposits
	return nil // Placeholder
}

// GetDataAnalysisSummary 获取数据分析摘要
// Method: tradingCompetitionAdmin_getDataAnalysisSummary
func (h *TradingCompetitionAdminHandler) GetDataAnalysisSummary(ctx context.Context, r *jsonrpc.Request, result *dto.GetDataAnalysisSummaryResponse) error {
	var params dto.GetDataAnalysisSummaryRequest
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionAdminSvc.GetDataAnalysisSummary(ctx, params)
	// return err
	return nil // Placeholder
}

// GetLeaderboard 获取排行榜数据
// Method: tradingCompetitionAdmin_getLeaderboard
func (h *TradingCompetitionAdminHandler) GetLeaderboard(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error { // Assuming result is common.ListResponse of DataAnalysisLeaderboardItemDTO
	var params struct {
		CompetitionID string       `json:"competition_id"`
		PoolID        string       `json:"pool_id"`
		Pagination    common.Page  `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// items, total, err := h.tradingCompetitionAdminSvc.GetLeaderboard(ctx, params.CompetitionID, params.PoolID, params.Pagination)
	// if err != nil { return err }
	// result.Items = items // Type assertion needed
	// result.Total = total
	return nil // Placeholder
}

// GetDetailedTransactions 获取明细交易数据
// Method: tradingCompetitionAdmin_getDetailedTransactions
func (h *TradingCompetitionAdminHandler) GetDetailedTransactions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error { // Assuming common.ListResponse of DataAnalysisTransactionItemDTO
	var params struct {
		CompetitionID string                                  `json:"competition_id"`
		PoolID        *string                                 `json:"pool_id,omitempty"`
		Pagination    common.Page                             `json:"pagination"`
		Filter        *dto.GetDetailedTransactionsRequestFilter `json:"filter,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// items, total, err := h.tradingCompetitionAdminSvc.GetDetailedTransactions(ctx, params.CompetitionID, params.PoolID, params.Pagination, params.Filter)
	// if err != nil { return err }
	// result.Items = items // Type assertion needed
	// result.Total = total
	return nil // Placeholder
}

// GetProjectTokenInfoByTwitter (utils_getProjectTokenInfoByTwitter) - 辅助接口
// Method: utils_getProjectTokenInfoByTwitter
func (h *TradingCompetitionAdminHandler) GetProjectTokenInfoByTwitter(ctx context.Context, r *jsonrpc.Request, result *dto.GetProjectTokenInfoByTwitterResponse) error {
	var params dto.GetProjectTokenInfoByTwitterRequest
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionAdminSvc.GetProjectTokenInfoByTwitter(ctx, params.TwitterHandle) // Or a utility service
	// return err
	return nil // Placeholder
}

```
--- 

## 4. API 设计 (HTTP - 前端视角)

本部分描述了B端管理后台前端通过HTTP调用的API。所有API均采用POST方法。

**通用约定:**
-   **Base URL**: `/api/v1/admin/trading-competitions` (示例，具体路径根据项目路由配置)
-   **HTTP Method**: `POST` (所有接口)
-   **认证**: 所有B端管理接口均需要认证，通过 `Authorization: Bearer <JWT_TOKEN>` 请求头传递。
-   **Content-Type**: 请求和响应体均为 `application/json`。
-   **错误处理**: 遵循项目统一的HTTP错误码和错误响应结构。
    -   `400 Bad Request`: 参数校验失败。
    -   `401 Unauthorized`: 未认证或认证失败。
    -   `403 Forbidden`: 无权限操作。
    -   `404 Not Found`: 资源不存在 (较少见，因为POST通常用于创建或操作，但如果依赖的内部资源找不到可能发生)。
    -   `500 Internal Server Error`: 服务器内部错误。
-   **请求体**: 所有先前通过路径参数或查询参数传递的数据，现在都作为字段包含在JSON请求体中。

### 活动管理

1.  **创建交易大赛**
    *   **Endpoint**: `/create`
    *   **Request Body**: `dto.CreateTradingCompetitionParamsDTO`
    *   **Response Body**: `dto.TradingCompetitionDetailDTO`
    *   **描述**: 创建一个新的交易大赛。后端自动设置创建者信息，状态默认为 `DRAFT`。

2.  **更新交易大赛**
    *   **Endpoint**: `/update`
    *   **Request Body**: `dto.UpdateTradingCompetitionParamsDTO` (其中 `competition_id` 为必填字段)
    *   **Response Body**: `dto.TradingCompetitionDetailDTO`
    *   **描述**: 更新指定ID的交易大赛信息。

3.  **保存交易大赛为草稿**
    *   **Endpoint**: `/save-draft`
    *   **Request Body**: `dto.CreateTradingCompetitionParamsDTO` 或 `dto.UpdateTradingCompetitionParamsDTO` (如果包含 `competition_id` 则为更新现有草稿)
    *   **Response Body**: `dto.TradingCompetitionDetailDTO`
    *   **描述**: 创建或更新一个草稿状态的交易大赛。

4.  **获取交易大赛列表**
    *   **Endpoint**: `/list`
    *   **Request Body**:
        ```json
        {
          "filter": {
            "name_like": "string, optional", // 活动名称模糊查询
            "status": "string, optional"     // 活动状态筛选 (e.g., "DRAFT", "ONGOING")
          },
          "pagination": { // 对应 common.Page
            "page_no": "int, optional, default: 1",
            "page_size": "int, optional, default: 10"
          }
        }
        ```
    *   **Response Body**: `common.ListResponse<dto.TradingCompetitionListItemDTO>`

5.  **获取交易大赛详情**
    *   **Endpoint**: `/detail`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**: `dto.TradingCompetitionDetailDTO`

6.  **发布交易大赛**
    *   **Endpoint**: `/publish`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**: `dto.BaseSuccessResponse` (或包含更新后状态的 `dto.TradingCompetitionDetailDTO`)
    *   **描述**: 发布一个交易大赛。后端校验预算、奖池等配置。

7.  **删除交易大赛**
    *   **Endpoint**: `/delete`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**: `dto.BaseSuccessResponse`
    *   **描述**: 软删除一个交易大赛。

### 预算与充值

8.  **为大赛充值代币**
    *   **Endpoint**: `/budget/deposit`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string",       // 交易大赛ID
          "token_address": "string",        // 代币合约地址
          "deposit_amount_str": "string"  // 充值数量 (大数字符串)
        }
        ```
    *   **Response Body**:
        ```json
        {
          "success": true,
          "message": "string, optional",
          "updated_deposit": { // 可选，返回更新后的充值信息
            "token_symbol": "string",
            "new_deposited_amount_str": "string"
          }
        }
        ```

9.  **获取大赛预算状态**
    *   **Endpoint**: `/budget/status`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**:
        ```json
        {
          "deposits": [ // 数组，每个元素对应 dto.DepositedTokenStatusDTO
            // ... (结构同之前 GET 响应)
          ]
        }
        ```
        (对应 `dto.GetCompetitionBudgetStatusResponse`)


### 数据分析

10. **获取数据分析摘要**
    *   **Endpoint**: `/analytics/summary`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string",             // 交易大赛ID
          "pool_id": "string, optional",          // 特定奖池ID
          "date_range": "string, optional, default: "ALL"" // 时间范围 ("7D", "30D", "ALL")
        }
        ```
    *   **Response Body**: `dto.GetDataAnalysisSummaryResponse`

11. **获取排行榜**
    *   **Endpoint**: `/analytics/leaderboard`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string", // 交易大赛ID
          "pool_id": "string",        // 奖池ID
          "pagination": {
            "page_no": "int, optional, default: 1",
            "page_size": "int, optional, default: 10"
          }
        }
        ```
    *   **Response Body**: `common.ListResponse<dto.DataAnalysisLeaderboardItemDTO>`

12. **获取明细交易数据**
    *   **Endpoint**: `/analytics/transactions`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string",             // 交易大赛ID
          "filter": {
            "pool_id": "string, optional",          // 特定奖池ID
            "user_address": "string, optional"    // 用户钱包地址筛选
          },
          "pagination": {
            "page_no": "int, optional, default: 1",
            "page_size": "int, optional, default: 10"
          }
        }
        ```
    *   **Response Body**: `common.ListResponse<dto.DataAnalysisTransactionItemDTO>`

### 辅助接口

13. **根据Twitter Handle获取项目代币信息**
    *   **Endpoint**: `/utils/project-info-by-twitter`
    *   **Request Body**:
        ```json
        {
          "twitter_handle": "string" // Twitter Handle
        }
        ```
    *   **Response Body**: `dto.GetProjectTokenInfoByTwitterResponse`
    *   **描述**: 用于B端配置项目信息时，通过Twitter Handle自动获取项目代币信息。

---