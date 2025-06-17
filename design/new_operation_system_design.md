# 运营体系 - 功能设计文档

## 1. 功能概述

本文档根据 `新需求/整理后的md/new-运营体系需求文档.md` 设计运营体系。该体系旨在通过每日签到、等级系统、积分商城、勋章墙等激励机制，提升用户活跃度、留存率和平台生态价值。

## 2. 数据库设计 (MySQL)

核心表结构用于存储用户运营数据、配置信息及相关流水。

```sql
-- user_operation_profiles: 存储用户的运营体系核心数据，如积分、经验、等级
CREATE TABLE `user_operation_profiles` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID (关联主用户表)',
  `current_points` BIGINT NOT NULL DEFAULT 0 COMMENT '当前可用积分',
  `total_points_earned` BIGINT NOT NULL DEFAULT 0 COMMENT '累计获得积分',
  `current_experience` BIGINT NOT NULL DEFAULT 0 COMMENT '当前经验值',
  `total_experience_earned` BIGINT NOT NULL DEFAULT 0 COMMENT '累计获得经验值',
  `current_level_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '当前等级ID (外键, level_configs.id)',
  `last_checkin_date` DATE DEFAULT NULL COMMENT '最后签到日期',
  `consecutive_checkin_days` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '连续签到天数',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_id` (`user_id`),
  INDEX `idx_level_id` (`current_level_id`),
  CONSTRAINT `fk_uop_level_id` FOREIGN KEY (`current_level_id`) REFERENCES `level_configs` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户运营档案表';

-- level_configs: 存储等级配置信息
CREATE TABLE `level_configs` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `level_number` INT UNSIGNED NOT NULL COMMENT '等级序号 (例如 1, 2, 3)',
  `level_name` VARCHAR(100) NOT NULL COMMENT '等级名称 (例如 "新手", "活跃达人")',
  `experience_threshold` BIGINT NOT NULL COMMENT '达到此等级所需经验阈值',
  `benefits_description_json` JSON DEFAULT NULL COMMENT '等级权益描述 (JSON格式, 例如 {"points_bonus_rate": 0.05, "exclusive_access": true})',
  `icon_url` VARCHAR(1024) DEFAULT NULL COMMENT '等级图标URL',
  `is_active` BOOLEAN NOT NULL DEFAULT TRUE COMMENT '是否启用',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_level_number` (`level_number`),
  INDEX `idx_experience_threshold` (`experience_threshold`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='等级配置表';

-- points_transactions: 存储积分流水记录
CREATE TABLE `points_transactions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID',
  `amount` BIGINT NOT NULL COMMENT '积分变动数量 (正数为增加, 负数为消耗)',
  `balance_after_tx` BIGINT NOT NULL COMMENT '交易后积分余额',
  `transaction_type` VARCHAR(50) NOT NULL COMMENT '交易类型 (例如 CHECK_IN, TASK_COMPLETE, MALL_REDEMPTION, ADMIN_ADJUST)',
  `description` VARCHAR(255) DEFAULT NULL COMMENT '交易描述',
  `related_entity_id` VARCHAR(255) DEFAULT NULL COMMENT '关联实体ID (例如 任务ID, 商品ID, 订单ID)',
  `related_entity_type` VARCHAR(50) DEFAULT NULL COMMENT '关联实体类型 (例如 TASK, PRODUCT, ORDER)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '交易时间',
  PRIMARY KEY (`id`),
  INDEX `idx_user_type_time` (`user_id`, `transaction_type`, `created_at`),
  INDEX `idx_user_time` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='积分流水表';

-- experience_transactions: 存储经验流水记录
CREATE TABLE `experience_transactions` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID',
  `amount` BIGINT NOT NULL COMMENT '经验变动数量',
  `balance_after_tx` BIGINT NOT NULL COMMENT '交易后经验余额',
  `transaction_type` VARCHAR(50) NOT NULL COMMENT '交易类型 (例如 CHECK_IN, TASK_COMPLETE, ADMIN_ADJUST)',
  `description` VARCHAR(255) DEFAULT NULL COMMENT '交易描述',
  `related_entity_id` VARCHAR(255) DEFAULT NULL COMMENT '关联实体ID',
  `related_entity_type` VARCHAR(50) DEFAULT NULL COMMENT '关联实体类型',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '交易时间',
  PRIMARY KEY (`id`),
  INDEX `idx_user_type_time` (`user_id`, `transaction_type`, `created_at`),
  INDEX `idx_user_time` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='经验流水表';

-- daily_checkin_configs: 存储每日签到配置 (B端配置)
CREATE TABLE `daily_checkin_configs` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID',
  `base_points_reward` INT NOT NULL DEFAULT 10 COMMENT '每日签到基础积分奖励',
  `base_experience_reward` INT NOT NULL DEFAULT 5 COMMENT '每日签到基础经验奖励',
  `consecutive_checkin_rules_json` JSON DEFAULT NULL COMMENT '连续签到额外奖励规则 (JSON数组, 例如 [{"days": 7, "extra_points": 50, "extra_experience": 20}])',
  `missed_checkin_compensation_enabled` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否启用补签功能',
  `missed_checkin_compensation_cost_points` INT DEFAULT NULL COMMENT '补签消耗积分 (如果启用)',
  `max_missed_checkin_days_allowed` INT DEFAULT 7 COMMENT '允许补签的最大天数',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='每日签到配置表';

-- daily_checkin_records: 存储用户每日签到记录
CREATE TABLE `daily_checkin_records` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID',
  `checkin_date` DATE NOT NULL COMMENT '签到日期',
  `points_rewarded` INT NOT NULL DEFAULT 0 COMMENT '本次签到奖励积分',
  `experience_rewarded` INT NOT NULL DEFAULT 0 COMMENT '本次签到奖励经验',
  `is_compensation` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否为补签',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '记录创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_date` (`user_id`, `checkin_date`),
  INDEX `idx_user_id_created` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户每日签到记录表';

-- badges: 存储勋章配置信息
CREATE TABLE `badges` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `name` VARCHAR(100) NOT NULL COMMENT '勋章名称',
  `description` VARCHAR(512) NOT NULL COMMENT '勋章描述',
  `icon_url` VARCHAR(1024) NOT NULL COMMENT '勋章图标URL',
  `rarity` VARCHAR(20) DEFAULT 'COMMON' COMMENT '稀有度 (COMMON, RARE, EPIC, LEGENDARY)',
  `acquisition_criteria_json` JSON NOT NULL COMMENT '获取条件 (JSON格式, 例如 {"type": "CONSECUTIVE_CHECKIN", "days": 30} or {"type": "LEVEL_REACHED", "level_id": 5})',
  `is_active` BOOLEAN NOT NULL DEFAULT TRUE COMMENT '是否启用',
  `display_order` INT NOT NULL DEFAULT 0 COMMENT '显示顺序',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='勋章配置表';

-- user_badges: 存储用户已获得的勋章
CREATE TABLE `user_badges` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID',
  `badge_id` BIGINT UNSIGNED NOT NULL COMMENT '勋章ID (外键, badges.id)',
  `achieved_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '获得勋章的时间',
  `is_worn` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否佩戴展示 (如果支持选择佩戴)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_badge` (`user_id`, `badge_id`),
  INDEX `idx_user_worn` (`user_id`, `is_worn`),
  CONSTRAINT `fk_ub_user_id` FOREIGN KEY (`user_id`) REFERENCES `user_operation_profiles` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE, -- 假设 user_id 在 user_operation_profiles 是唯一的
  CONSTRAINT `fk_ub_badge_id` FOREIGN KEY (`badge_id`) REFERENCES `badges` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户勋章表';

-- mall_products: 存储积分商城商品信息
CREATE TABLE `mall_products` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键',
  `name` VARCHAR(255) NOT NULL COMMENT '商品名称',
  `description` TEXT DEFAULT NULL COMMENT '商品描述',
  `product_type` VARCHAR(50) NOT NULL COMMENT '商品类型 (VIRTUAL_GOODS, PHYSICAL_GOODS, PLATFORM_BENEFIT, COUPON)',
  `image_url` VARCHAR(1024) DEFAULT NULL COMMENT '商品图片URL',
  `points_cost` INT UNSIGNED NOT NULL COMMENT '兑换所需积分',
  `stock_quantity` INT DEFAULT -1 COMMENT '库存数量 (-1表示无限库存)',
  `redeem_limit_per_user` INT DEFAULT -1 COMMENT '每用户限兑次数 (-1表示无限制)',
  `redeem_limit_total` INT DEFAULT -1 COMMENT '总兑换次数限制 (-1表示无限制)',
  `required_level_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '最低兑换等级ID (外键, level_configs.id)',
  `valid_from` DATETIME(3) DEFAULT NULL COMMENT '可兑换开始时间',
  `valid_to` DATETIME(3) DEFAULT NULL COMMENT '可兑换结束时间',
  `is_active` BOOLEAN NOT NULL DEFAULT TRUE COMMENT '是否上架/启用',
  `display_order` INT NOT NULL DEFAULT 0 COMMENT '显示顺序',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_type_active_points` (`product_type`, `is_active`, `points_cost`),
  CONSTRAINT `fk_mp_level_id` FOREIGN KEY (`required_level_id`) REFERENCES `level_configs` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='积分商城商品表';

-- mall_redemption_orders: 存储积分商城兑换订单
CREATE TABLE `mall_redemption_orders` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '唯一ID，主键 (订单号)',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户ID',
  `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID (外键, mall_products.id)',
  `points_spent` INT UNSIGNED NOT NULL COMMENT '消耗积分数量',
  `quantity` INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '兑换数量',
  `status` VARCHAR(20) NOT NULL DEFAULT 'PROCESSING' COMMENT '订单状态 (PROCESSING:处理中, COMPLETED:已完成, FAILED:失败, CANCELED:已取消)',
  `delivery_info_json` JSON DEFAULT NULL COMMENT '配送信息 (JSON, 针对实体商品)',
  `notes` VARCHAR(512) DEFAULT NULL COMMENT '备注 (例如失败原因)',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '兑换时间',
  `updated_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '最后更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_user_product_status` (`user_id`, `product_id`, `status`),
  INDEX `idx_user_status_time` (`user_id`, `status`, `created_at`),
  CONSTRAINT `fk_mro_user_id` FOREIGN KEY (`user_id`) REFERENCES `user_operation_profiles` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `fk_mro_product_id` FOREIGN KEY (`product_id`) REFERENCES `mall_products` (`id`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='积分商城兑换订单表';

-- (Optional) invitation_rewards_log: 如果邀请系统的奖励与运营体系紧密相关且需要单独记录
-- CREATE TABLE `invitation_rewards_log` (
--   `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
--   `inviter_user_id` VARCHAR(255) NOT NULL COMMENT '邀请人用户ID',
--   `invitee_user_id` VARCHAR(255) NOT NULL COMMENT '被邀请人用户ID',
--   `reward_type` VARCHAR(50) NOT NULL COMMENT '奖励类型 (POINTS, EXPERIENCE, BADGE)',
--   `points_awarded` INT DEFAULT NULL,
--   `experience_awarded` INT DEFAULT NULL,
--   `badge_id_awarded` BIGINT UNSIGNED DEFAULT NULL,
--   `description` VARCHAR(255) DEFAULT NULL,
--   `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
--   PRIMARY KEY (`id`),
--   INDEX `idx_inviter_invitee` (`inviter_user_id`, `invitee_user_id`),
--   CONSTRAINT `fk_irl_badge_id` FOREIGN KEY (`badge_id_awarded`) REFERENCES `badges` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
-- ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='邀请奖励日志表 (运营体系相关)';

```

## 3. API 设计 (JSON-RPC)

遵循 `taskon-server` (或相关服务) 的API设计规范。

### 3.1 DTOs (C-端: 用户视角)
Package `dto` (位于 `internal/application/dto/opsuser/` 或类似路径)

```go
package opsuserdto // or a more general dto package

import (
	"taskon/common" // 假设 common.Page 和 common.ListResponse 在此
	"time"
)

// LevelBenefitDTO 等级权益信息
type LevelBenefitDTO struct {
	Description string `json:"description"` // 权益描述
	// 可根据实际权益定义更多字段，例如 points_bonus_rate, exclusive_access_key
}

// UserLevelInfoDTO 用户等级信息
type UserLevelInfoDTO struct {
	LevelID             string            `json:"level_id"`               // 等级ID (数字字符串)
	LevelNumber         int               `json:"level_number"`           // 等级序号
	LevelName           string            `json:"level_name"`             // 等级名称
	IconURL             *string           `json:"icon_url,omitempty"`     // 等级图标URL
	ExperienceThreshold int64             `json:"experience_threshold"`   // 当前等级所需经验 (或下一级阈值)
	Benefits            []LevelBenefitDTO `json:"benefits,omitempty"`     // 当前等级权益
}

// UserOperationProfileDTO 用户运营档案
type UserOperationProfileDTO struct {
	UserID                   string            `json:"user_id"`                     // 用户ID
	CurrentPoints            int64             `json:"current_points"`              // 当前积分
	TotalPointsEarned        int64             `json:"total_points_earned"`         // 累计获得积分
	CurrentExperience        int64             `json:"current_experience"`          // 当前经验值
	TotalExperienceEarned    int64             `json:"total_experience_earned"`     // 累计获得经验值
	CurrentLevel             *UserLevelInfoDTO `json:"current_level,omitempty"`     // 当前等级信息
	NextLevelExperienceNeeded *int64           `json:"next_level_experience_needed,omitempty"` // 距离下一级所需经验
	LastCheckinDate          *string           `json:"last_checkin_date,omitempty"` // YYYY-MM-DD
	ConsecutiveCheckinDays   int               `json:"consecutive_checkin_days"`    // 连续签到天数
}

// CheckinStatusDTO 签到状态
type CheckinStatusDTO struct {
	TodayCheckedIn          bool              `json:"today_checked_in"`            // 今天是否已签到
	ConsecutiveCheckinDays  int               `json:"consecutive_checkin_days"`    // 当前连续签到天数
	LastCheckinDate         *string           `json:"last_checkin_date,omitempty"` // YYYY-MM-DD
	CheckinCalendar         []CheckinDayDTO   `json:"checkin_calendar"`            // 本月签到日历 (或近N天)
	BasePointsReward        int               `json:"base_points_reward"`          // 签到基础积分
	BaseExperienceReward    int               `json:"base_experience_reward"`      // 签到基础经验
	ConsecutiveRewardRules  []ConsecutiveRewardRuleDTO `json:"consecutive_reward_rules,omitempty"` // 连续签到奖励规则
	MissedCheckinAvailable  bool              `json:"missed_checkin_available"`    // 是否可补签
	MissedCheckinCostPoints *int              `json:"missed_checkin_cost_points,omitempty"` // 补签消耗积分
}

// CheckinDayDTO 签到日历中的一天
type CheckinDayDTO struct {
	Date      string `json:"date"`       // YYYY-MM-DD
	CheckedIn bool   `json:"checked_in"` // 当天是否已签到
}

// ConsecutiveRewardRuleDTO 连续签到奖励规则
type ConsecutiveRewardRuleDTO struct {
	Days            int `json:"days"`                       // 需要连续签到天数
	ExtraPoints     int `json:"extra_points,omitempty"`     // 额外积分奖励
	ExtraExperience int `json:"extra_experience,omitempty"` // 额外经验奖励
	// BadgeID string `json:"badge_id,omitempty"` // 可选：奖励特定勋章
}

// PerformCheckinResponseDTO 执行签到操作的响应
type PerformCheckinResponseDTO struct {
	Success              bool    `json:"success"`
	Message              string  `json:"message"`
	PointsRewarded       int     `json:"points_rewarded"`
	ExperienceRewarded   int     `json:"experience_rewarded"`
	CurrentPoints        int64   `json:"current_points"`
	CurrentExperience    int64   `json:"current_experience"`
	ConsecutiveCheckinDays int   `json:"consecutive_checkin_days"`
	LeveledUp            bool    `json:"leveled_up"`                       // 是否升级
	NewLevel             *UserLevelInfoDTO `json:"new_level,omitempty"`    // 如果升级，新的等级信息
	AwardedBadges        []BadgeDTO `json:"awarded_badges,omitempty"`      // 本次操作获得的勋章
}

// BadgeDTO 勋章信息
type BadgeDTO struct {
	ID                 string  `json:"id"`                   // 勋章ID (数字字符串)
	Name               string  `json:"name"`                 // 勋章名称
	Description        string  `json:"description"`          // 勋章描述
	IconURL            string  `json:"icon_url"`             // 勋章图标URL
	Rarity             *string `json:"rarity,omitempty"`     // 稀有度
	AchievedAt         *string `json:"achieved_at,omitempty"`// 获得时间 (ISO8601, 如果是用户已获得的勋章)
	AcquisitionCriteria string `json:"acquisition_criteria,omitempty"` // 获取条件描述 (用于展示未获得的勋章)
	IsWorn             *bool   `json:"is_worn,omitempty"`    // 是否佩戴 (如果是用户已获得的勋章)
}

// MallProductListItemDTO 商城商品列表项
type MallProductListItemDTO struct {
	ID            string  `json:"id"`             // 商品ID (数字字符串)
	Name          string  `json:"name"`           // 商品名称
	ProductType   string  `json:"product_type"`   // 商品类型
	ImageURL      *string `json:"image_url,omitempty"`// 商品图片URL
	PointsCost    int     `json:"points_cost"`    // 所需积分
	StockRemaining *int   `json:"stock_remaining,omitempty"` // 剩余库存 (-1表示无限)
	IsRedeemable  bool    `json:"is_redeemable"`  // 当前用户是否可兑换 (基于等级、次数限制等)
}

// MallProductDetailDTO 商城商品详情
type MallProductDetailDTO struct {
	ID                       string  `json:"id"`                         // 商品ID
	Name                     string  `json:"name"`                       // 商品名称
	Description              *string `json:"description,omitempty"`      // 商品描述
	ProductType              string  `json:"product_type"`               // 商品类型
	ImageURL                 *string `json:"image_url,omitempty"`        // 商品图片URL
	PointsCost               int     `json:"points_cost"`                // 所需积分
	StockQuantity            *int    `json:"stock_quantity,omitempty"`   // 库存 (-1无限)
	RedeemLimitPerUser       *int    `json:"redeem_limit_per_user,omitempty"` // 每用户限兑 (-1无限)
	RedeemLimitTotal         *int    `json:"redeem_limit_total,omitempty"`    // 总限兑 (-1无限)
	UserRedeemedCount        int     `json:"user_redeemed_count"`        // 当前用户已兑换次数
	RequiredLevelName        *string `json:"required_level_name,omitempty"`// 最低兑换等级名称
	ValidFrom                *string `json:"valid_from,omitempty"`       // 可兑换开始时间 (ISO8601)
	ValidTo                  *string `json:"valid_to,omitempty"`         // 可兑换结束时间 (ISO8601)
	IsActive                 bool    `json:"is_active"`                  // 是否上架
	IsCurrentUserRedeemable  bool    `json:"is_current_user_redeemable"` // 当前用户是否可兑换
}

// RedeemMallProductResponseDTO 兑换商城商品响应
type RedeemMallProductResponseDTO struct {
	Success       bool    `json:"success"`
	Message       string  `json:"message"`
	OrderID       *string `json:"order_id,omitempty"` // 兑换订单ID (数字字符串)
	CurrentPoints int64   `json:"current_points"`   // 兑换后剩余积分
}

// RedemptionOrderDTO 用户兑换订单信息
type RedemptionOrderDTO struct {
	ID            string  `json:"id"`             // 订单ID
	ProductName   string  `json:"product_name"`   // 商品名称
	ProductImageURL *string `json:"product_image_url,omitempty"` // 商品图片
	PointsSpent   int     `json:"points_spent"`   // 消耗积分
	Quantity      int     `json:"quantity"`       // 兑换数量
	Status        string  `json:"status"`         // 订单状态
	CreatedAt     string  `json:"created_at"`     // 兑换时间 (ISO8601)
	Notes         *string `json:"notes,omitempty"`// 备注
}

// PointsTransactionDTO 积分流水项
type PointsTransactionDTO struct {
	ID                string  `json:"id"`
	Amount            int64   `json:"amount"`             // 正负表示增减
	BalanceAfterTx    int64   `json:"balance_after_tx"`   // 交易后余额
	TransactionType   string  `json:"transaction_type"`   // 交易类型
	Description       *string `json:"description,omitempty"`// 描述
	CreatedAt         string  `json:"created_at"`         // 交易时间 (ISO8601)
}

// ExperienceTransactionDTO 经验流水项
type ExperienceTransactionDTO struct {
	ID                string  `json:"id"`
	Amount            int64   `json:"amount"`
	BalanceAfterTx    int64   `json:"balance_after_tx"`
	TransactionType   string  `json:"transaction_type"`
	Description       *string `json:"description,omitempty"`
	CreatedAt         string  `json:"created_at"`
}

```

### 3.2 DTOs (B-端: 管理后台)
Package `dto` (位于 `internal/application/dto/opsadmin/` 或类似路径)

```go
package opsadmindto

import (
	"taskon/common"
	opsuserdto "taskon/internal/application/dto/opsuser" // 引用C端DTO
)

// LevelConfigParamsDTO 等级配置参数 (创建/更新)
type LevelConfigParamsDTO struct {
	ID                   *string                          `json:"id,omitempty"` // 更新时提供 (数字字符串)
	LevelNumber          int                              `json:"level_number" binding:"required,min=1"`
	LevelName            string                           `json:"level_name" binding:"required,max=100"`
	ExperienceThreshold  int64                            `json:"experience_threshold" binding:"required,min=0"`
	BenefitsDescriptionJson *string                       `json:"benefits_description_json,omitempty"` // JSON string
	IconURL              *string                          `json:"icon_url,omitempty,url"`
	IsActive             *bool                            `json:"is_active,omitempty"` // 默认为true
}

// LevelConfigDTO 等级配置详情 (B端展示)
type LevelConfigDTO struct {
	ID                   string            `json:"id"`
	LevelNumber          int               `json:"level_number"`
	LevelName            string            `json:"level_name"`
	ExperienceThreshold  int64             `json:"experience_threshold"`
	BenefitsDescriptionJson *string        `json:"benefits_description_json,omitempty"`
	IconURL              *string           `json:"icon_url,omitempty"`
	IsActive             bool              `json:"is_active"`
	CreatedAt            string            `json:"created_at"`
	UpdatedAt            string            `json:"updated_at"`
}

// BadgeConfigParamsDTO 勋章配置参数 (创建/更新)
type BadgeConfigParamsDTO struct {
	ID                   *string  `json:"id,omitempty"` // 更新时提供 (数字字符串)
	Name                 string   `json:"name" binding:"required,max=100"`
	Description          string   `json:"description" binding:"required,max=512"`
	IconURL              string   `json:"icon_url" binding:"required,url"`
	Rarity               *string  `json:"rarity,omitempty,oneof=COMMON RARE EPIC LEGENDARY"` // 默认为 COMMON
	AcquisitionCriteriaJson string `json:"acquisition_criteria_json" binding:"required,json"` // JSON string
	IsActive             *bool    `json:"is_active,omitempty"` // 默认为true
	DisplayOrder         *int     `json:"display_order,omitempty"` // 默认为0
}

// BadgeConfigDTO 勋章配置详情 (B端展示)
type BadgeConfigDTO struct {
	ID                   string  `json:"id"`
	Name                 string  `json:"name"`
	Description          string  `json:"description"`
	IconURL              string  `json:"icon_url"`
	Rarity               string  `json:"rarity"`
	AcquisitionCriteriaJson string `json:"acquisition_criteria_json"`
	IsActive             bool    `json:"is_active"`
	DisplayOrder         int     `json:"display_order"`
	CreatedAt            string  `json:"created_at"`
	UpdatedAt            string  `json:"updated_at"`
}

// DailyCheckinConfigDTO 每日签到配置 (B端展示/更新)
type DailyCheckinConfigDTO struct {
	BasePointsReward        int                                       `json:"base_points_reward" binding:"min=0"`
	BaseExperienceReward    int                                       `json:"base_experience_reward" binding:"min=0"`
	ConsecutiveCheckinRulesJson *string                                `json:"consecutive_checkin_rules_json,omitempty,json"` // JSON array string
	MissedCheckinCompensationEnabled bool                             `json:"missed_checkin_compensation_enabled"`
	MissedCheckinCompensationCostPoints *int                          `json:"missed_checkin_compensation_cost_points,omitempty,min=0"`
	MaxMissedCheckinDaysAllowed *int                                  `json:"max_missed_checkin_days_allowed,omitempty,min=0"`
	UpdatedAt               string                                    `json:"updated_at,omitempty"` // 只读
}

// MallProductConfigParamsDTO 商城商品配置参数 (创建/更新)
type MallProductConfigParamsDTO struct {
	ID                   *string  `json:"id,omitempty"` // 更新时提供 (数字字符串)
	Name                 string   `json:"name" binding:"required,max=255"`
	Description          *string  `json:"description,omitempty"`
	ProductType          string   `json:"product_type" binding:"required,oneof=VIRTUAL_GOODS PHYSICAL_GOODS PLATFORM_BENEFIT COUPON"`
	ImageURL             *string  `json:"image_url,omitempty,url"`
	PointsCost           int      `json:"points_cost" binding:"required,min=0"`
	StockQuantity        *int     `json:"stock_quantity,omitempty"` // -1 无限
	RedeemLimitPerUser   *int     `json:"redeem_limit_per_user,omitempty"` // -1 无限
	RedeemLimitTotal     *int     `json:"redeem_limit_total,omitempty"` // -1 无限
	RequiredLevelID      *string  `json:"required_level_id,omitempty"` // 等级ID (数字字符串)
	ValidFrom            *string  `json:"valid_from,omitempty"` // ISO8601
	ValidTo              *string  `json:"valid_to,omitempty"`   // ISO8601
	IsActive             *bool    `json:"is_active,omitempty"` // 默认为true
	DisplayOrder         *int     `json:"display_order,omitempty"` // 默认为0
}

// MallProductConfigDTO 商城商品配置详情 (B端展示)
type MallProductConfigDTO struct {
	ID                   string  `json:"id"`
	Name                 string  `json:"name"`
	Description          *string `json:"description,omitempty"`
	ProductType          string  `json:"product_type"`
	ImageURL             *string `json:"image_url,omitempty"`
	PointsCost           int     `json:"points_cost"`
	StockQuantity        *int    `json:"stock_quantity,omitempty"`
	CurrentStock         *int    `json:"current_stock,omitempty"` // 实时库存 (计算得出)
	RedeemLimitPerUser   *int    `json:"redeem_limit_per_user,omitempty"`
	RedeemLimitTotal     *int    `json:"redeem_limit_total,omitempty"`
	TotalRedeemedCount   int     `json:"total_redeemed_count"` // 总已兑换数量
	RequiredLevelID      *string `json:"required_level_id,omitempty"`
	RequiredLevelName    *string `json:"required_level_name,omitempty"`
	ValidFrom            *string `json:"valid_from,omitempty"`
	ValidTo              *string `json:"valid_to,omitempty"`
	IsActive             bool    `json:"is_active"`
	DisplayOrder         int     `json:"display_order"`
	CreatedAt            string  `json:"created_at"`
	UpdatedAt            string  `json:"updated_at"`
}

// AdminAdjustUserResourceParamsDTO 管理员调整用户资源参数
type AdminAdjustUserResourceParamsDTO struct {
	UserID        string  `json:"user_id" binding:"required"`
	PointsAmount  *int64  `json:"points_amount,omitempty"`  // 正数增加，负数减少
	ExperienceAmount *int64 `json:"experience_amount,omitempty"` // 正数增加
	Description   string  `json:"description" binding:"required,max=255"`
}

// AdminGrantUserBadgeParamsDTO 管理员授予用户勋章参数
type AdminGrantUserBadgeParamsDTO struct {
	UserID      string `json:"user_id" binding:"required"`
	BadgeID     string `json:"badge_id" binding:"required"` // 勋章ID (数字字符串)
	Description *string `json:"description,omitempty,max=255"`
}
```

### 3.3 JSON-RPC Go Handler Signatures (C-端: 用户)
Package `handler` (位于 `api/jsonrpc/handlers/opsuser/` 或类似路径)

```go
package opsuserhandler

import (
	"context"

	"taskon/common"
	"taskon/common/jsonrpc"
	"taskon/internal/application/dto/opsuser" // C端DTOs
	// "taskon/internal/service/opsuser" // C端服务层接口
)

// UserOperationHandler 封装了运营体系C端用户相关的JSON-RPC方法
type UserOperationHandler struct {
	// opsUserSvc opsuser.Service
}

// GetMyProfile 获取当前用户的运营档案 (积分、经验、等级等)。
// Method: opsUser_getMyProfile
// Params: {}
// Response: opsuser.UserOperationProfileDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块"
func (h *UserOperationHandler) GetMyProfile(ctx context.Context, r *jsonrpc.Request, result *opsuser.UserOperationProfileDTO) error {
	// userIDFromCtx := ctx.Value("userID").(string)
	// *result, err = h.opsUserSvc.GetMyProfile(ctx, userIDFromCtx)
	// return err
	return nil // Placeholder
}

// GetCheckinStatus 获取用户的签到状态和日历。
// Method: opsUser_getCheckinStatus
// Params: {}
// Response: opsuser.CheckinStatusDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 每日签到模块"
func (h *UserOperationHandler) GetCheckinStatus(ctx context.Context, r *jsonrpc.Request, result *opsuser.CheckinStatusDTO) error {
	// userIDFromCtx := ctx.Value("userID").(string)
	// *result, err = h.opsUserSvc.GetCheckinStatus(ctx, userIDFromCtx)
	// return err
	return nil // Placeholder
}

// PerformCheckin 用户执行每日签到。
// Method: opsUser_performCheckin
// Params: {}
// Response: opsuser.PerformCheckinResponseDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到"
func (h *UserOperationHandler) PerformCheckin(ctx context.Context, r *jsonrpc.Request, result *opsuser.PerformCheckinResponseDTO) error {
	// userIDFromCtx := ctx.Value("userID").(string)
	// *result, err = h.opsUserSvc.PerformCheckin(ctx, userIDFromCtx)
	// return err
	return nil // Placeholder
}

// PerformMissedCheckin 用户执行补签 (如果启用且符合条件)。
// Method: opsUser_performMissedCheckin
// Params: { "date_to_compensate": "YYYY-MM-DD" }
// Response: opsuser.PerformCheckinResponseDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 每日签到模块 -> 补签功能 (可选)"
func (h *UserOperationHandler) PerformMissedCheckin(ctx context.Context, r *jsonrpc.Request, result *opsuser.PerformCheckinResponseDTO) error {
	var params struct {
		DateToCompensate string `json:"date_to_compensate"` // YYYY-MM-DD
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// *result, err = h.opsUserSvc.PerformMissedCheckin(ctx, userIDFromCtx, params.DateToCompensate)
	// return err
	return nil // Placeholder
}

// ListMyBadges 获取用户已获得的勋章列表，以及可选的未获得勋章信息。
// Method: opsUser_listMyBadges
// Params: { "include_unachieved": "bool, optional, default: false" }
// Response: common.ListResponse (of opsuser.BadgeDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块 -> 我的勋章"
func (h *UserOperationHandler) ListMyBadges(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		IncludeUnachieved bool `json:"include_unachieved,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// items, total, err := h.opsUserSvc.ListMyBadges(ctx, userIDFromCtx, params.IncludeUnachieved)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// WearBadge 用户选择佩戴某个勋章 (如果支持)。
// Method: opsUser_wearBadge
// Params: { "badge_id": "string" }
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：勋章墙 -> 佩戴展示" (可选)
func (h *UserOperationHandler) WearBadge(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params struct {
		BadgeID string `json:"badge_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// err := h.opsUserSvc.WearBadge(ctx, userIDFromCtx, params.BadgeID)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

// ListMallProducts 获取积分商城商品列表。
// Method: opsUser_listMallProducts
// Params: { "category": "string, optional", "sort_by": "points_asc|points_desc|newest, optional", "pagination": common.Page }
// Response: common.ListResponse (of opsuser.MallProductListItemDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"
func (h *UserOperationHandler) ListMallProducts(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Category  *string     `json:"category,omitempty"`
		SortBy    *string     `json:"sort_by,omitempty"`
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty filter/sort for pagination
	}
	// userIDFromCtx := ctx.Value("userID").(string) // May need for redeemable status
	// items, total, err := h.opsUserSvc.ListMallProducts(ctx, userIDFromCtx, params)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// GetMallProductDetail 获取积分商城商品详情。
// Method: opsUser_getMallProductDetail
// Params: { "product_id": "string" }
// Response: opsuser.MallProductDetailDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 商品详情页"
func (h *UserOperationHandler) GetMallProductDetail(ctx context.Context, r *jsonrpc.Request, result *opsuser.MallProductDetailDTO) error {
	var params struct {
		ProductID string `json:"product_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// userIDFromCtx := ctx.Value("userID").(string) // For user-specific redeemable status
	// *result, err = h.opsUserSvc.GetMallProductDetail(ctx, userIDFromCtx, params.ProductID)
	// return err
	return nil // Placeholder
}

// RedeemMallProduct 用户兑换积分商城商品。
// Method: opsUser_redeemMallProduct
// Params: { "product_id": "string", "quantity": "int, default:1" }
// Response: opsuser.RedeemMallProductResponseDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：积分商城"
func (h *UserOperationHandler) RedeemMallProduct(ctx context.Context, r *jsonrpc.Request, result *opsuser.RedeemMallProductResponseDTO) error {
	var params struct {
		ProductID string `json:"product_id"`
		Quantity  *int   `json:"quantity,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	qty := 1
	if params.Quantity != nil { qty = *params.Quantity }

	// userIDFromCtx := ctx.Value("userID").(string)
	// *result, err = h.opsUserSvc.RedeemMallProduct(ctx, userIDFromCtx, params.ProductID, qty)
	// return err
	return nil // Placeholder
}

// ListMyRedemptionOrders 获取用户的兑换记录。
// Method: opsUser_listMyRedemptionOrders
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsuser.RedemptionOrderDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 兑换记录"
func (h *UserOperationHandler) ListMyRedemptionOrders(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty for pagination
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// items, total, err := h.opsUserSvc.ListMyRedemptionOrders(ctx, userIDFromCtx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// ListMyPointsTransactions 获取用户的积分流水。
// Method: opsUser_listMyPointsTransactions
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsuser.PointsTransactionDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块 -> 我的积分 (账单)"
func (h *UserOperationHandler) ListMyPointsTransactions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty for pagination
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// items, total, err := h.opsUserSvc.ListMyPointsTransactions(ctx, userIDFromCtx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// ListMyExperienceTransactions 获取用户的经验流水。
// Method: opsUser_listMyExperienceTransactions
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsuser.ExperienceTransactionDTO)
// Requirement Source: Implied by "我的经验与等级".
func (h *UserOperationHandler) ListMyExperienceTransactions(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty for pagination
	}
	// userIDFromCtx := ctx.Value("userID").(string)
	// items, total, err := h.opsUserSvc.ListMyExperienceTransactions(ctx, userIDFromCtx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

```

### 3.4 JSON-RPC Go Handler Signatures (B-端: 管理后台)
Package `handler` (位于 `api/jsonrpc/handlers/opsadmin/` 或类似路径)

```go
package opsadminhandler

import (
	"context"

	"taskon/common"
	"taskon/common/jsonrpc"
	"taskon/internal/application/dto/opsadmin" // B端DTOs
	// "taskon/internal/service/opsadmin" // B端服务层接口
)

// AdminOperationHandler 封装了运营体系B端管理相关的JSON-RPC方法
type AdminOperationHandler struct {
	// opsAdminSvc opsadmin.Service
}

// UpsertLevelConfig 创建或更新等级配置。
// Method: opsAdmin_upsertLevelConfig
// Params: opsadmin.LevelConfigParamsDTO
// Response: opsadmin.LevelConfigDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块 -> 等级定义与经验值配置 (B端)"
func (h *AdminOperationHandler) UpsertLevelConfig(ctx context.Context, r *jsonrpc.Request, result *opsadmin.LevelConfigDTO) error {
	var params opsadmin.LevelConfigParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.opsAdminSvc.UpsertLevelConfig(ctx, params)
	// return err
	return nil // Placeholder
}

// ListLevelConfigs 获取等级配置列表。
// Method: opsAdmin_listLevelConfigs
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsadmin.LevelConfigDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块"
func (h *AdminOperationHandler) ListLevelConfigs(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty
	}
	// items, total, err := h.opsAdminSvc.ListLevelConfigs(ctx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// DeleteLevelConfig 删除等级配置。
// Method: opsAdmin_deleteLevelConfig
// Params: { "level_id": "string" }
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块"
func (h *AdminOperationHandler) DeleteLevelConfig(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params struct {
		LevelID string `json:"level_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.opsAdminSvc.DeleteLevelConfig(ctx, params.LevelID)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

// UpsertBadgeConfig 创建或更新勋章配置。
// Method: opsAdmin_upsertBadgeConfig
// Params: opsadmin.BadgeConfigParamsDTO
// Response: opsadmin.BadgeConfigDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块 -> 勋章设计与配置 (B端)"
func (h *AdminOperationHandler) UpsertBadgeConfig(ctx context.Context, r *jsonrpc.Request, result *opsadmin.BadgeConfigDTO) error {
	var params opsadmin.BadgeConfigParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.opsAdminSvc.UpsertBadgeConfig(ctx, params)
	// return err
	return nil // Placeholder
}

// ListBadgeConfigs 获取勋章配置列表。
// Method: opsAdmin_listBadgeConfigs
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsadmin.BadgeConfigDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块"
func (h *AdminOperationHandler) ListBadgeConfigs(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty
	}
	// items, total, err := h.opsAdminSvc.ListBadgeConfigs(ctx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// DeleteBadgeConfig 删除勋章配置。
// Method: opsAdmin_deleteBadgeConfig
// Params: { "badge_id": "string" }
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块"
func (h *AdminOperationHandler) DeleteBadgeConfig(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params struct {
		BadgeID string `json:"badge_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.opsAdminSvc.DeleteBadgeConfig(ctx, params.BadgeID)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

// GetDailyCheckinConfig 获取每日签到配置。
// Method: opsAdmin_getDailyCheckinConfig
// Params: {}
// Response: opsadmin.DailyCheckinConfigDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到 -> B端可配置"
func (h *AdminOperationHandler) GetDailyCheckinConfig(ctx context.Context, r *jsonrpc.Request, result *opsadmin.DailyCheckinConfigDTO) error {
	// *result, err = h.opsAdminSvc.GetDailyCheckinConfig(ctx)
	// return err
	return nil // Placeholder
}

// UpdateDailyCheckinConfig 更新每日签到配置。
// Method: opsAdmin_updateDailyCheckinConfig
// Params: opsadmin.DailyCheckinConfigDTO (partial updates allowed based on DTO fields)
// Response: opsadmin.DailyCheckinConfigDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到 -> B端可配置"
func (h *AdminOperationHandler) UpdateDailyCheckinConfig(ctx context.Context, r *jsonrpc.Request, result *opsadmin.DailyCheckinConfigDTO) error {
	var params opsadmin.DailyCheckinConfigDTO // Assuming all fields are optional for update
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.opsAdminSvc.UpdateDailyCheckinConfig(ctx, params)
	// return err
	return nil // Placeholder
}

// UpsertMallProductConfig 创建或更新商城商品配置。
// Method: opsAdmin_upsertMallProductConfig
// Params: opsadmin.MallProductConfigParamsDTO
// Response: opsadmin.MallProductConfigDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> B端配置"
func (h *AdminOperationHandler) UpsertMallProductConfig(ctx context.Context, r *jsonrpc.Request, result *opsadmin.MallProductConfigDTO) error {
	var params opsadmin.MallProductConfigParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.opsAdminSvc.UpsertMallProductConfig(ctx, params)
	// return err
	return nil // Placeholder
}

// ListMallProductConfigs 获取商城商品配置列表。
// Method: opsAdmin_listMallProductConfigs
// Params: { "pagination": common.Page }
// Response: common.ListResponse (of opsadmin.MallProductConfigDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"
func (h *AdminOperationHandler) ListMallProductConfigs(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty
	}
	// items, total, err := h.opsAdminSvc.ListMallProductConfigs(ctx, params.Pagination)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// DeleteMallProductConfig 删除商城商品配置。
// Method: opsAdmin_deleteMallProductConfig
// Params: { "product_id": "string" }
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"
func (h *AdminOperationHandler) DeleteMallProductConfig(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params struct {
		ProductID string `json:"product_id"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.opsAdminSvc.DeleteMallProductConfig(ctx, params.ProductID)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

// ListUserRedemptionOrders 管理员查看用户兑换订单列表。
// Method: opsAdmin_listUserRedemptionOrders
// Params: { "user_id": "string, optional", "product_id": "string, optional", "status": "string, optional", "pagination": common.Page }
// Response: common.ListResponse (of opsuser.RedemptionOrderDTO)
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 库存管理 (B端)" (implies viewing orders)
func (h *AdminOperationHandler) ListUserRedemptionOrders(ctx context.Context, r *jsonrpc.Request, result *common.ListResponse) error {
	var params struct {
		UserID     *string     `json:"user_id,omitempty"`
		ProductID  *string     `json:"product_id,omitempty"`
		Status     *string     `json:"status,omitempty"`
		Pagination common.Page `json:"pagination"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		// allow empty filters
	}
	// items, total, err := h.opsAdminSvc.ListUserRedemptionOrders(ctx, params)
	// if err != nil { return err }
	// result.Items = items; result.Total = total
	return nil // Placeholder
}

// UpdateRedemptionOrderStatus 管理员更新兑换订单状态。
// Method: opsAdmin_updateRedemptionOrderStatus
// Params: { "order_id": "string", "new_status": "string (COMPLETED, FAILED, CANCELED)", "notes": "string, optional" }
// Response: opsuser.RedemptionOrderDTO
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：积分商城 -> 实体商品进入待发货流程"
func (h *AdminOperationHandler) UpdateRedemptionOrderStatus(ctx context.Context, r *jsonrpc.Request, result *opsuser.RedemptionOrderDTO) error {
	var params struct {
		OrderID   string  `json:"order_id"`
		NewStatus string  `json:"new_status"`
		Notes     *string `json:"notes,omitempty"`
	}
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// *result, err = h.opsAdminSvc.UpdateRedemptionOrderStatus(ctx, params.OrderID, params.NewStatus, params.Notes)
	// return err
	return nil // Placeholder
}


// AdjustUserResources 管理员调整用户积分/经验。
// Method: opsAdmin_adjustUserResources
// Params: opsadmin.AdminAdjustUserResourceParamsDTO
// Response: common.BaseSuccessResponse
// Requirement Source: 新需求/整理后的md/new-运营体系需求文档.md - "四、业务规则 -> BR-OS-010 作弊行为处理" (implies manual adjustment capability)
func (h *AdminOperationHandler) AdjustUserResources(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params opsadmin.AdminAdjustUserResourceParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.opsAdminSvc.AdjustUserResources(ctx, params)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

// GrantBadgeToUser 管理员手动授予用户勋章。
// Method: opsAdmin_grantBadgeToUser
// Params: opsadmin.AdminGrantUserBadgeParamsDTO
// Response: common.BaseSuccessResponse
// Requirement Source: Implied for administrative purposes, e.g. correcting errors or special awards.
func (h *AdminOperationHandler) GrantBadgeToUser(ctx context.Context, r *jsonrpc.Request, result *common.BaseSuccessResponse) error {
	var params opsadmin.AdminGrantUserBadgeParamsDTO
	if err := jsonrpc.UnmarshalParams(r.Params, &params); err != nil {
		return err
	}
	// err := h.opsAdminSvc.GrantBadgeToUser(ctx, params)
	// result.Success = err == nil; if err != nil {result.Message = err.Error()}
	return nil // Placeholder
}

```

## 4. API 设计 (HTTP - 前端视角)

### 4.1 C-端 (用户) HTTP API
**通用约定:**
-   **Base URL**: `/api/v1/ops/user` (示例)
-   **HTTP Method**: `POST`
-   **认证**: 用户 JWT
-   **Content-Type**: `application/json`

1.  **获取我的运营档案**
    *   **Endpoint**: `/profile`
    *   **Request Body**: `{}`
    *   **Response Body**: `opsuser.UserOperationProfileDTO`
    *   **描述**: 获取当前用户的积分、经验、等级等信息。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块"

2.  **获取签到状态**
    *   **Endpoint**: `/checkin/status`
    *   **Request Body**: `{}`
    *   **Response Body**: `opsuser.CheckinStatusDTO`
    *   **描述**: 获取用户当天的签到状态和签到日历。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 每日签到模块"

3.  **执行签到**
    *   **Endpoint**: `/checkin/perform`
    *   **Request Body**: `{}`
    *   **Response Body**: `opsuser.PerformCheckinResponseDTO`
    *   **描述**: 用户执行每日签到操作。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到"

4.  **执行补签**
    *   **Endpoint**: `/checkin/compensate`
    *   **Request Body**: `{ "date_to_compensate": "YYYY-MM-DD" }`
    *   **Response Body**: `opsuser.PerformCheckinResponseDTO`
    *   **描述**: 用户执行补签操作。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 每日签到模块 -> 补签功能 (可选)"

5.  **获取我的勋章列表**
    *   **Endpoint**: `/badges/list`
    *   **Request Body**: `{ "include_unachieved": false }` (optional)
    *   **Response Body**: `common.ListResponse<opsuser.BadgeDTO>`
    *   **描述**: 获取用户已获得的勋章和可选的未获得勋章。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块 -> 我的勋章"

6.  **佩戴勋章**
    *   **Endpoint**: `/badges/wear`
    *   **Request Body**: `{ "badge_id": "string" }`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **描述**: 用户选择佩戴某个勋章。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：勋章墙 -> 佩戴展示" (可选)

7.  **获取商城商品列表**
    *   **Endpoint**: `/mall/products/list`
    *   **Request Body**: `{ "category": "string, optional", "sort_by": "string, optional", "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsuser.MallProductListItemDTO>`
    *   **描述**: 获取积分商城商品列表。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"

8.  **获取商城商品详情**
    *   **Endpoint**: `/mall/products/detail`
    *   **Request Body**: `{ "product_id": "string" }`
    *   **Response Body**: `opsuser.MallProductDetailDTO`
    *   **描述**: 获取特定积分商城商品详情。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 商品详情页"

9.  **兑换商城商品**
    *   **Endpoint**: `/mall/products/redeem`
    *   **Request Body**: `{ "product_id": "string", "quantity": 1 }`
    *   **Response Body**: `opsuser.RedeemMallProductResponseDTO`
    *   **描述**: 用户使用积分兑换商城商品。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：积分商城"

10. **获取我的兑换记录**
    *   **Endpoint**: `/mall/orders/list`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsuser.RedemptionOrderDTO>`
    *   **描述**: 获取用户的积分商城兑换记录。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 兑换记录"

11. **获取我的积分流水**
    *   **Endpoint**: `/points/transactions`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsuser.PointsTransactionDTO>`
    *   **描述**: 获取用户的积分变动历史。
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 用户账户模块 -> 我的积分 (账单)"

12. **获取我的经验流水**
    *   **Endpoint**: `/experience/transactions`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsuser.ExperienceTransactionDTO>`
    *   **描述**: 获取用户的经验变动历史。
    *   **Requirement Source**: Implied by "我的经验与等级".

### 4.2 B-端 (管理后台) HTTP API
**通用约定:**
-   **Base URL**: `/api/v1/ops/admin` (示例)
-   **HTTP Method**: `POST`
-   **认证**: 管理员 JWT
-   **Content-Type**: `application/json`

1.  **创建/更新等级配置**
    *   **Endpoint**: `/levels/upsert`
    *   **Request Body**: `opsadmin.LevelConfigParamsDTO`
    *   **Response Body**: `opsadmin.LevelConfigDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块 -> 等级定义与经验值配置 (B端)"

2.  **获取等级配置列表**
    *   **Endpoint**: `/levels/list`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsadmin.LevelConfigDTO>`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块"

3.  **删除等级配置**
    *   **Endpoint**: `/levels/delete`
    *   **Request Body**: `{ "level_id": "string" }`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 等级系统模块"

4.  **创建/更新勋章配置**
    *   **Endpoint**: `/badges/upsert`
    *   **Request Body**: `opsadmin.BadgeConfigParamsDTO`
    *   **Response Body**: `opsadmin.BadgeConfigDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块 -> 勋章设计与配置 (B端)"

5.  **获取勋章配置列表**
    *   **Endpoint**: `/badges/list`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsadmin.BadgeConfigDTO>`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块"

6.  **删除勋章配置**
    *   **Endpoint**: `/badges/delete`
    *   **Request Body**: `{ "badge_id": "string" }`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 勋章系统模块"

7.  **获取每日签到配置**
    *   **Endpoint**: `/checkin-config/get`
    *   **Request Body**: `{}`
    *   **Response Body**: `opsadmin.DailyCheckinConfigDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到 -> B端可配置"

8.  **更新每日签到配置**
    *   **Endpoint**: `/checkin-config/update`
    *   **Request Body**: `opsadmin.DailyCheckinConfigDTO`
    *   **Response Body**: `opsadmin.DailyCheckinConfigDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：每日签到 -> B端可配置"

9.  **创建/更新商城商品配置**
    *   **Endpoint**: `/mall/products/upsert`
    *   **Request Body**: `opsadmin.MallProductConfigParamsDTO`
    *   **Response Body**: `opsadmin.MallProductConfigDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> B端配置"

10. **获取商城商品配置列表**
    *   **Endpoint**: `/mall/products/list-configs`
    *   **Request Body**: `{ "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsadmin.MallProductConfigDTO>`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"

11. **删除商城商品配置**
    *   **Endpoint**: `/mall/products/delete`
    *   **Request Body**: `{ "product_id": "string" }`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块"

12. **获取用户兑换订单列表 (管理)**
    *   **Endpoint**: `/mall/orders/list-all`
    *   **Request Body**: `{ "user_id": "string, optional", "product_id": "string, optional", "status": "string, optional", "pagination": common.Page }`
    *   **Response Body**: `common.ListResponse<opsuser.RedemptionOrderDTO>`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 1. 功能地图 -> 积分商城模块 -> 库存管理 (B端)"

13. **更新兑换订单状态 (管理)**
    *   **Endpoint**: `/mall/orders/update-status`
    *   **Request Body**: `{ "order_id": "string", "new_status": "string", "notes": "string, optional" }`
    *   **Response Body**: `opsuser.RedemptionOrderDTO`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "三、功能设计 -> 2. 功能描述模板 -> 功能点：积分商城 -> 实体商品进入待发货流程"

14. **调整用户积分/经验 (管理)**
    *   **Endpoint**: `/users/adjust-resources`
    *   **Request Body**: `opsadmin.AdminAdjustUserResourceParamsDTO`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **Requirement Source**: 新需求/整理后的md/new-运营体系需求文档.md - "四、业务规则 -> BR-OS-010 作弊行为处理"

15. **授予用户勋章 (管理)**
    *   **Endpoint**: `/users/grant-badge`
    *   **Request Body**: `opsadmin.AdminGrantUserBadgeParamsDTO`
    *   **Response Body**: `common.BaseSuccessResponse`
    *   **Requirement Source**: Implied for administrative needs.

---
**注意**:
- 上述 DTO 和 Handler 中的 `common.Page`, `common.ListResponse`, `common.BaseSuccessResponse` 假定为项目中已有的通用结构。
- 服务层接口的具体实现未在此定义。
- 错误处理、具体的认证和授权逻辑需要在实际实现中细化。
- "Requirement Source" 指向了 `新需求/整理后的md/new-运营体系需求文档.md` 中的相关章节。
- JSON 字段中的 `binding` tag 是 `gin-gonic` 框架的示例，实际使用根据项目技术栈调整。
- ID 字段如 `level_id`, `badge_id`, `product_id` 在 DTO 中通常为字符串形式，以兼容不同数字类型或 UUID。
- 邀请系统的具体API未在此详述，假设其为独立模块或通过任务系统间接影响运营体系的积分/经验。如果运营体系需要直接处理邀请奖励，则需补充相关表和API。

</rewritten_file>
