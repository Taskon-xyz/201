# 交易大赛 - C端功能设计文档

## 1. 功能概述

本文档根据 `新需求/整理后的md/new-交易大赛-C端.md` 设计交易大赛C端（用户参与方）功能。用户可以查看活动详情、规则、连接钱包、进行Swap交易、查看个人数据和排行榜，并在活动结束后领取奖励。

## 2. 数据库设计 (MySQL)

C端功能主要依赖于B端创建的活动数据。大部分数据读取自B端已设计的表 (`trading_competitions`, `trading_competition_projects`, `trading_competition_pools`, `trading_competition_user_volume`)。C端可能需要额外的表来跟踪用户针对特定活动的奖励领取状态，如果这不由专门的奖励服务处理。

```sql
-- trading_competition_user_rewards: 存储用户在交易大赛中获得的奖励及其领取状态
-- 注意: 此表的设计假设奖励计算和发放不由 taskon-server 直接处理链上交互，
-- 而是由 taskon-actions 或专门的奖励/清算服务负责更新此表的状态。
CREATE TABLE `trading_competition_user_rewards` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `competition_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的交易大赛ID (外键, trading_competitions.id)',
  `pool_id` BIGINT UNSIGNED NOT NULL COMMENT '关联的奖池ID (外键, trading_competition_pools.id)',
  `user_wallet_address` VARCHAR(255) NOT NULL COMMENT '用户钱包地址',
  `reward_token_address` VARCHAR(255) NOT NULL COMMENT '奖励代币的合约地址',
  `reward_token_symbol` VARCHAR(50) NOT NULL COMMENT '奖励代币的符号',
  `reward_token_decimals` INT NOT NULL COMMENT '奖励代币的精度',
  `reward_amount_raw` VARCHAR(78) NOT NULL COMMENT '用户获得的奖励数量 (以原始精度的字符串形式存储)',
  `claim_status` VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT '奖励领取状态 (PENDING:待处理, CLAIMABLE:可领取, CLAIMED:已领取, FAILED:领取失败, EXPIRED:已过期)',
  `claimed_tx_hash` VARCHAR(255) DEFAULT NULL COMMENT '用户领取奖励的交易哈希 (如果适用)',
  `claimed_at` DATETIME(3) DEFAULT NULL COMMENT '用户实际领取奖励的时间',
  `calculated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '奖励计算完成并记录到此表的时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '记录最后更新时间',
  `notes` VARCHAR(512) DEFAULT NULL COMMENT '备注，例如领取失败原因、过期原因等',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_pool_user_token_reward` (`pool_id`, `user_wallet_address`, `reward_token_address` COMMENT '确保用户在同一奖池同种代币的奖励记录唯一'),
  INDEX `idx_comp_user_claim_status` (`competition_id`, `user_wallet_address`, `claim_status` COMMENT '按活动、用户和领取状态查询奖励'),
  CONSTRAINT `fk_tcur_competition_id` FOREIGN KEY (`competition_id`) REFERENCES `trading_competitions` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `fk_tcur_pool_id` FOREIGN KEY (`pool_id`) REFERENCES `trading_competition_pools` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='交易大赛用户奖励表，记录用户的获奖和领取情况';
```

**注意**: 奖励的计算和分配逻辑可能由一个独立的后端数据处理服务完成，该服务会填充 `trading_competition_user_rewards` 表或通知 `taskon-actions` 服务进行处理。

## 3. API 设计 (JSON-RPC)

遵循 `taskon-actions/guideline.md`。C端API主要用于数据展示和触发特定操作（如领奖）。

**DTOs (C-端)**
Package `dto` (位于 `internal/application/dto/` 或类似路径)

```go
package dto

import (
	"taskon/common" // 假设 common.Page 和 common.ListResponse 在此
)

// CompetitionProjectInfoDTO 定义了C端展示的项目信息
type CompetitionProjectInfoDTO struct {
	ProjectName          string `json:"project_name"`                   // 项目名称
	ProjectLogoURL       string `json:"project_logo_url"`               // 项目Logo URL
	ProjectTwitterHandle string `json:"project_twitter_handle,omitempty"` // 项目Twitter Handle (可选)
	ProjectWebsiteURL    string `json:"project_website_url,omitempty"`    // 项目官网URL (可选)
}

// CompetitionPoolRuleDTO 定义了C端展示的奖池规则信息
type CompetitionPoolRuleDTO struct {
	PoolID                   string `json:"pool_id"`                      // 奖池ID (数字字符串)
	PoolName                 string `json:"pool_name"`                    // 奖池名称
	PoolType                 string `json:"pool_type"`                    // 奖池类型 ("BASIC", "SPECIFIED")
	RewardTokenSymbol        string `json:"reward_token_symbol"`          // 奖励代币符号
	TotalRewardAmountStr     string `json:"total_reward_amount_str"`      // 奖池总奖金 (格式化字符串, 例如 "1,000 USDT")
	Description              string `json:"description,omitempty"`        // 奖池规则描述 (例如 "Top 10 by volume share the rewards")
	EffectiveTradingPairsDesc string `json:"effective_trading_pairs_desc"` // 有效交易对描述 (例如 "Any pair on Uniswap V3" 或 "WETH/USDT on PancakeSwap")
	WinnerEligibilityDesc    string `json:"winner_eligibility_desc,omitempty"` // 获奖资格描述 (例如 "Trading Volume >= 100 USD" or "Top 10 Rank")
}

// CompetitionDetailForUserDTO 定义了C端用户视角的活动详情
type CompetitionDetailForUserDTO struct {
	ID                       string                        `json:"id"`                                 // 活动ID (数字字符串)
	Name                     string                        `json:"name"`                               // 活动名称
	BannerPCURL              string                        `json:"banner_pc_url,omitempty"`            // PC端Banner图片URL
	BannerMobileURL          string                        `json:"banner_mobile_url,omitempty"`        // Mobile端Banner图片URL
	StartTime                string                        `json:"start_time"`                         // 活动开始时间 (ISO8601 UTC 字符串)
	EndTime                  string                        `json:"end_time"`                           // 活动结束时间 (ISO8601 UTC 字符串)
	CurrentStatus            string                        `json:"current_status"`                     // 活动当前状态 ("UPCOMING", "ONGOING", "ENDED", "CALCULATING_RESULTS", "RESULTS_ANNOUNCED")
	RemainingTimeSeconds     *int64                        `json:"remaining_time_seconds,omitempty"`  // 剩余时间 (秒, 如果活动未结束)
	DotAnimationEnabled      bool                          `json:"dot_animation_enabled"`              // 是否启用点状动画
	PageStyleConfig          *TradingCompetitionPageStyleDTO `json:"page_style_config,omitempty"`        // 页面样式配置 (可复用B端DTO或定义类似结构)
	NetworkID                string                        `json:"network_id"`                         // 链ID
	DexID                    string                        `json:"dex_id"`                             // DEX ID
	ParticipatingProjects    []CompetitionProjectInfoDTO   `json:"participating_projects,omitempty"`   // 参与项目列表
	RulesSummary             string                        `json:"rules_summary,omitempty"`            // 总规则概述 (HTML或Markdown格式)
	PoolRules                []CompetitionPoolRuleDTO      `json:"pool_rules"`                         // 各奖池规则列表
	TotalPrizeValueUSDStr    string                        `json:"total_prize_value_usd_str,omitempty"`// 活动总奖金价值 (USD, 格式化字符串)
	TotalTradingVolumeUSDStr string                        `json:"total_trading_volume_usd_str,omitempty"`// 活动当前总交易量 (USD, 格式化字符串)
	TotalParticipants        int                           `json:"total_participants,omitempty"`       // 活动当前总参与人数
	UserSpecificData         *UserCompetitionOverallDataDTO `json:"user_specific_data,omitempty"`    // 当前用户的总体数据 (如果用户已连接钱包且活动进行中或已结束)
}

// UserCompetitionDataDTO 定义了用户在单个奖池中的数据
type UserCompetitionDataDTO struct {
	PoolID                  string  `json:"pool_id"`                             // 奖池ID (数字字符串)
	PoolName                string  `json:"pool_name"`                           // 奖池名称
	MyTradingVolumeUSDStr   string  `json:"my_trading_volume_usd_str"`           // 我的交易量 (USD, 格式化字符串)
	MyVolumeSharePercentStr *string `json:"my_volume_share_percent_str,omitempty"` // 我的交易量占比 (例如 "0.55%", 可选)
	MyRank                  *int    `json:"my_rank,omitempty"`                     // 我的排名 (可选, 活动进行中或结束后)
	MyEstimatedRewardStr    *string `json:"my_estimated_reward_str,omitempty"`   // 我的预计奖励 (例如 "10.5 USDT", 活动进行中, 可选)
}

// UserCompetitionOverallDataDTO 聚合用户在一个活动中的所有奖池数据
type UserCompetitionOverallDataDTO struct {
	DataPerPool               []UserCompetitionDataDTO `json:"data_per_pool"`                             // 用户在各奖池的数据
	TotalMyVolumeUSDStr       string                   `json:"total_my_volume_usd_str,omitempty"`         // 用户在活动中的总交易额 (USD, 格式化字符串)
	TotalMyEstimatedRewardStr string                   `json:"total_my_estimated_reward_str,omitempty"`   // 用户在活动中的总预估奖励 (格式化字符串)
}

// LeaderboardEntryDTO 定义了排行榜条目
type LeaderboardEntryDTO struct {
	Rank                   int     `json:"rank"`                                // 排名
	UserDisplayAddress     string  `json:"user_display_address"`                // 用户显示地址 (例如 "0x1234...abcd" 或 ENS 名称)
	TradingVolumeUSDStr    string  `json:"trading_volume_usd_str"`              // 交易量 (USD, 格式化字符串)
	VolumeSharePercentStr  *string `json:"volume_share_percent_str,omitempty"`  // 交易量占比 (可选)
	EstimatedRewardStr     *string `json:"estimated_reward_str,omitempty"`      // 预计奖励 (可选, 格式化字符串, 活动进行中时显示)
	IsCurrentUser          bool    `json:"is_current_user,omitempty"`           // 是否为当前查询用户 (如果API调用时提供了user_wallet_address)
}

// UserRewardItemDTO 定义了用户可领取的单个奖励项
type UserRewardItemDTO struct {
	RewardRecordID      string  `json:"reward_record_id"`                  // 奖励记录ID (来自 trading_competition_user_rewards.id, 数字字符串)
	PoolID              string  `json:"pool_id"`                           // 奖池ID (数字字符串)
	PoolName            string  `json:"pool_name"`                         // 奖池名称
	RewardTokenSymbol   string  `json:"reward_token_symbol"`               // 奖励代币符号
	RewardAmountStr     string  `json:"reward_amount_str"`                 // 奖励数量 (格式化字符串)
	ClaimStatus         string  `json:"claim_status"`                      // 领取状态 ("PENDING", "CLAIMABLE", "CLAIMED", "PENDING_CONFIRMATION", "FAILED", "EXPIRED")
	ClaimedTxHash       *string `json:"claimed_tx_hash,omitempty"`         // 领取交易哈希 (可选)
	ClaimableFromTime   *string `json:"claimable_from_time,omitempty"`     // 可领取开始时间 (ISO8601 UTC, 可选)
	ErrorMessage        *string `json:"error_message,omitempty"`           // 领取失败时的错误信息 (可选)
}

// UserActivityResultDTO 定义了用户活动结束后的结果
type UserActivityResultDTO struct {
	OverallStatus  string                `json:"overall_status"`                    // 用户总体结果 ("WON", "NOT_WON", "PENDING_CALCULATION", "NO_PARTICIPATION")
	Rewards        []UserRewardItemDTO   `json:"rewards,omitempty"`                 // 用户获得的奖励列表 (如果WON)
	FinalData      []UserCompetitionDataDTO `json:"final_data,omitempty"`            // 用户最终的各奖池数据 (排名、交易量等)
}

// TokenInfoDTO (通用, 可复用)
type TokenInfoDTO struct {
	Address  string `json:"address"`           // 代币合约地址
	Symbol   string `json:"symbol"`            // 代币符号
	Decimals int    `json:"decimals"`          // 代币精度
	LogoURI  string `json:"logo_uri,omitempty"`  // 代币Logo的URI (可选)
}

// SwapContextDTO 定义了Swap组件所需的上下文信息
type SwapContextDTO struct {
	NetworkID              string         `json:"network_id"`                         // 活动指定的链ID
	DexID                  string         `json:"dex_id"`                             // 活动指定的DEX ID
	AllowedSwapFromTokens  []TokenInfoDTO `json:"allowed_swap_from_tokens,omitempty"` // 允许的Swap源代币列表 (根据奖池规则, 可选)
	AllowedSwapToTokens    []TokenInfoDTO `json:"allowed_swap_to_tokens,omitempty"`   // 允许的Swap目标代币列表 (根据奖池规则, 可选)
	DefaultFromToken       *TokenInfoDTO  `json:"default_from_token,omitempty"`       // 默认Swap源代币 (可选)
	DefaultToToken         *TokenInfoDTO  `json:"default_to_token,omitempty"`         // 默认Swap目标代币 (可选)
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

// TradingCompetitionUserHandler 封装了交易大赛C端用户相关的JSON-RPC方法
type TradingCompetitionUserHandler struct {
	// tradingCompetitionUserSvc service.TradingCompetitionUserService // 实际项目中会注入相应的服务层接口
}

// NewTradingCompetitionUserHandler 创建 TradingCompetitionUserHandler 实例
// func NewTradingCompetitionUserHandler(svc service.TradingCompetitionUserService) *TradingCompetitionUserHandler {
//	 return &TradingCompetitionUserHandler{tradingCompetitionUserSvc: svc}
// }

// GetCompetitionDetail 获取C端用户视角的活动详情。
// Method: tradingCompetitionUser_getCompetitionDetail
// Params: { "competition_id": "string", "user_wallet_address": "string" (optional) }
// Response: dto.CompetitionDetailForUserDTO
func (h *TradingCompetitionUserHandler) GetCompetitionDetail(ctx context.Context, r *jsonrpc.Request, result *dto.CompetitionDetailForUserDTO) error {
	var params struct {
		CompetitionID     string  `json:"competition_id"`
		UserWalletAddress *string `json:"user_wallet_address,omitempty"` // 可选，用于关联和返回用户的特定数据
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err // 框架会处理为JSON-RPC错误
	}
	// *result, err = h.tradingCompetitionUserSvc.GetCompetitionDetail(ctx, params.CompetitionID, params.UserWalletAddress)
	// return err
	return nil // Placeholder for actual service call
}

// GetPoolLeaderboard 获取指定奖池的排行榜。
// Method: tradingCompetitionUser_getPoolLeaderboard
// Params: { "competition_id": "string", "pool_id": "string", "pagination": common.Page, "user_wallet_address": "string" (optional) }
// Response: common.ListResponse (of dto.LeaderboardEntryDTO)
func (h *TradingCompetitionUserHandler) GetPoolLeaderboard(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		CompetitionID     string      `json:"competition_id"`
		PoolID            string      `json:"pool_id"`
		Pagination        common.Page `json:"pagination"` // common.Page 包含 PageNo 和 PageSize
		UserWalletAddress *string     `json:"user_wallet_address,omitempty"` // 可选，用于在排行榜中高亮或定位当前用户
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// items, total, err := h.tradingCompetitionUserSvc.GetPoolLeaderboard(ctx, params.CompetitionID, params.PoolID, params.Pagination, params.UserWalletAddress)
	// if err != nil { return err }
	// result.Items = items // 需要进行类型断言或使用泛型ListResponse
	// result.Total = total
	return nil // Placeholder
}

// GetMyDataForCompetition 获取用户在活动中各奖池的个人数据统计。
// Method: tradingCompetitionUser_getMyDataForCompetition
// Params: { "competition_id": "string", "user_wallet_address": "string" }
// Response: dto.UserCompetitionOverallDataDTO
func (h *TradingCompetitionUserHandler) GetMyDataForCompetition(ctx context.Context, r *jsonrpc.Request, result *dto.UserCompetitionOverallDataDTO) error {
	var params struct {
		CompetitionID     string `json:"competition_id"`
		UserWalletAddress string `json:"user_wallet_address"` // 必须提供用户钱包地址以查询个人数据
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionUserSvc.GetMyDataForCompetition(ctx, params.CompetitionID, params.UserWalletAddress)
	// return err
	return nil // Placeholder
}

// GetMyActivityResult 获取用户活动结束后的最终结果和奖励情况。
// Method: tradingCompetitionUser_getMyActivityResult
// Params: { "competition_id": "string", "user_wallet_address": "string" }
// Response: dto.UserActivityResultDTO
func (h *TradingCompetitionUserHandler) GetMyActivityResult(ctx context.Context, r *jsonrpc.Request, result *dto.UserActivityResultDTO) error {
	var params struct {
		CompetitionID     string `json:"competition_id"`
		UserWalletAddress string `json:"user_wallet_address"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionUserSvc.GetMyActivityResult(ctx, params.CompetitionID, params.UserWalletAddress)
	// return err
	return nil // Placeholder
}

// ClaimReward 领取单个奖励。
// 注意: 此方法逻辑上更可能属于 taskon-actions 或专门的奖励服务，此处为示例。
// Method: tradingCompetitionUser_claimReward (or a more specific like actions_claimTradingCompetitionReward)
// Params: { "reward_record_id": "string", "user_wallet_address": "string" }
// Response: dto.UserRewardItemDTO (updated status of the claimed item)
func (h *TradingCompetitionUserHandler) ClaimReward(ctx context.Context, r *jsonrpc.Request, result *dto.UserRewardItemDTO) error {
	var params struct {
		RewardRecordID    string `json:"reward_record_id"` // 对应 trading_competition_user_rewards.id
		UserWalletAddress string `json:"user_wallet_address"`
		// CompetitionID and PoolID might be implicit via RewardRecordID in service layer, or passed for additional validation
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionUserSvc.ClaimReward(ctx, params.RewardRecordID, params.UserWalletAddress)
	// return err
	return nil // Placeholder
}

// ClaimAllRewards 批量领取用户在活动中的所有可领奖励。
// 注意: 此方法逻辑上更可能属于 taskon-actions 或专门的奖励服务。
// Method: tradingCompetitionUser_claimAllRewards (or actions_claimAllTradingCompetitionRewards)
// Params: { "competition_id": "string", "user_wallet_address": "string" }
// Response: struct { Results []dto.UserRewardItemDTO; OverallSuccess bool; Message string (optional) }
func (h *TradingCompetitionUserHandler) ClaimAllRewards(ctx context.Context, r *jsonrpc.Request, result *struct{ Results []dto.UserRewardItemDTO; OverallSuccess bool; Message string `json:"message,omitempty"` }) error {
	var params struct {
		CompetitionID     string `json:"competition_id"`
		UserWalletAddress string `json:"user_wallet_address"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// updatedRewards, overallSuccess, message, err := h.tradingCompetitionUserSvc.ClaimAllRewards(ctx, params.CompetitionID, params.UserWalletAddress)
	// if err != nil { return err }
	// result.Results = updatedRewards
	// result.OverallSuccess = overallSuccess
	// result.Message = message
	return nil // Placeholder
}

// GetSwapContext 获取Swap组件初始化所需的上下文信息 (可选辅助接口)。
// Method: tradingCompetitionUser_getSwapContext
// Params: { "competition_id": "string", "pool_id": "string" (optional) }
// Response: dto.SwapContextDTO
func (h *TradingCompetitionUserHandler) GetSwapContext(ctx context.Context, r *jsonrpc.Request, result *dto.SwapContextDTO) error {
	var params struct {
		CompetitionID string  `json:"competition_id"`
		PoolID        *string `json:"pool_id,omitempty"` // 可选，如果需要针对特定奖池的交易对上下文
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.tradingCompetitionUserSvc.GetSwapContext(ctx, params.CompetitionID, params.PoolID)
	// return err
	return nil // Placeholder
}

```

## 4. API 设计 (HTTP - 前端视角)

本部分描述了C端用户通过HTTP调用的API。所有API均采用POST方法，主要用于活动展示、用户数据查询和奖励领取。

**通用约定:**
-   **Base URL**: `/api/v1/trading-competitions` (示例，具体路径根据项目路由配置)
-   **HTTP Method**: `POST` (所有接口)
-   **认证**: 
    -   公开可访问接口 (如获取活动详情、排行榜) 通常不强制认证，但如果请求中包含用户认证信息，可以用于个性化展示。
    -   需要用户身份的接口 (如获取个人数据、领取奖励) 需要通过 `Authorization: Bearer <USER_JWT_TOKEN>` 请求头传递用户认证信息。
-   **Content-Type**: 请求和响应体均为 `application/json`。
-   **错误处理**: 遵循项目统一的HTTP错误码和错误响应结构。
-   **请求体**: 所有先前通过路径参数或查询参数传递的数据，现在都作为字段包含在JSON请求体中。

### 活动浏览与参与

1.  **获取交易大赛详情 (C端用户视角)**
    *   **Endpoint**: `/detail`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string", // 交易大赛ID
          "user_wallet_address": "string, optional" // 用户钱包地址，用于关联返回用户特定数据
        }
        ```
    *   **Response Body**: `dto.CompetitionDetailForUserDTO`
    *   **描述**: 获取单个交易大赛的详细信息，供用户查看。

2.  **获取奖池排行榜 (C端用户视角)**
    *   **Endpoint**: `/pool/leaderboard`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string", // 交易大赛ID
          "pool_id": "string",        // 奖池ID
          "pagination": { // 对应 common.Page
            "page_no": "int, optional, default: 1",
            "page_size": "int, optional, default: 10"
          },
          "user_wallet_address": "string, optional" // 用户钱包地址，用于在排行榜中高亮或标记当前用户
        }
        ```
    *   **Response Body**: `common.ListResponse<dto.LeaderboardEntryDTO>`

3.  **获取我的活动数据 (需认证)**
    *   **Endpoint**: `/my-data`
    *   **Headers**: `Authorization: Bearer <USER_JWT_TOKEN>`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**: `dto.UserCompetitionOverallDataDTO`
    *   **描述**: 获取当前认证用户在指定交易大赛中所有奖池的个人数据统计。

4.  **获取我的活动结果与奖励 (需认证)**
    *   **Endpoint**: `/my-results`
    *   **Headers**: `Authorization: Bearer <USER_JWT_TOKEN>`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**: `dto.UserActivityResultDTO`
    *   **描述**: 获取当前认证用户在活动结束后的最终结果，包括奖励列表。

### 奖励领取 (需认证)

5.  **领取单个奖励**
    *   **Endpoint**: `/rewards/claim`
    *   **Headers**: `Authorization: Bearer <USER_JWT_TOKEN>`
    *   **Request Body**:
        ```json
        {
          "reward_record_id": "string" // 奖励记录ID (来自 trading_competition_user_rewards.id)
        }
        ```
    *   **Response Body**: `dto.UserRewardItemDTO` (返回更新后的奖励项状态)
    *   **描述**: 用户领取指定的单个奖励。

6.  **批量领取活动所有可领奖励**
    *   **Endpoint**: `/claim-all-rewards`
    *   **Headers**: `Authorization: Bearer <USER_JWT_TOKEN>`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string" // 交易大赛ID
        }
        ```
    *   **Response Body**:
        ```json
        {
          "overall_success": true,
          "message": "string, optional",
          "results": [ // 数组，每个元素对应 dto.UserRewardItemDTO (更新后的状态)
            // ... (同 dto.UserRewardItemDTO)
          ]
        }
        ```
    *   **描述**: 用户尝试批量领取在指定活动中所有状态为 `CLAIMABLE` 的奖励。

### 辅助接口

7.  **获取Swap组件上下文信息**
    *   **Endpoint**: `/swap-context`
    *   **Request Body**:
        ```json
        {
          "competition_id": "string",             // 交易大赛ID
          "pool_id": "string, optional"           // 奖池ID。如果提供，则返回针对该特定奖池的上下文
        }
        ```
    *   **Response Body**: `dto.SwapContextDTO`
    *   **描述**: 为前端Swap组件提供初始化所需的上下文信息。

--- 