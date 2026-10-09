# 05 · 用户画像与记忆（User Profile & Memory）

> 让「决策官」从"每次从零开始"升级为"越用越懂你"。
> 本文件定义一个结构化用户画像卡（UserProfile），规定它如何从 MCP 数据构建、如何在对话中更新、如何跨轮记忆。
> 这是把"单轮推荐助手"变成"有记忆的推荐系统"的关键模块。

## 一、UserProfile Schema

```json
{
  "profile_version": "1.0",
  "updated_at": "2026-10-09T16:55:00+08:00",

  "taste_vector": {
    "鸡肉": 0.8, "牛肉": 0.4, "鱼": 0.3, "素": 0.1,
    "辣": 0.5, "咖啡": 0.7, "甜品": 0.4, "小食_薯翅": 0.7
  },
  "dislikes": [],                      // 忌口/过敏/明确不吃，硬约束
  "price_anchor": { "typical": 20, "p25": 14.9, "p75": 37 },
  "preferred_scene": "到店自取",
  "frequent_items": [                  // 来自 order-list 的高频
    { "productName": "板烧鸡腿堡", "count": 3 },
    { "productName": "双层吉士随心配", "count": 4 }
  ],
  "home_store": { "storeCode": "1990262001", "name": "南京徐庄路快捷小站" },
  "health_goal": null,                 // null/控热量(红线kcal)/扛饿

  "recent_feedback": [                 // 短期反馈记忆，影响下一轮
    { "ts": "...", "signal": "too_expensive", "target": "安格斯厚牛堡" },
    { "ts": "...", "signal": "bored", "target": "板烧鸡腿堡" }
  ],
  "exploration_state": { "epsilon": 0.2, "tried_new": ["龙焰鸡腿堡"] }
}
```

## 二、画像构建（从 MCP 冷数据 → 画像）

首次或画像过期时，用真实接口重建：

1. `order-list` → 统计 `frequent_items`、`price_anchor`（订单金额分位）、`preferred_scene`、`home_store`。
2. 从高频商品反推 `taste_vector`：如多单板烧/双层吉士 → 鸡肉/牛肉权重高；常点薯/翅 → `小食_薯翅` 高；常点麦咖啡 → `咖啡` 高。
3. `query-my-account` → 若有将过期积分，置一个 `benefit_flag`。
4. `query-my-coupons` → 构建券池快照（含到期时间），供 R3 召回与兜底。

> 真实样例（当前演示账号）：高频板烧鸡腿堡/双层吉士随心配、price_anchor≈{p25:14.9, typical:20, p75:37}、home_store=徐庄路快捷小站、咖啡偏好明显。

## 三、画像更新与记忆（读 → 用 → 回写）

每一轮对话遵循 **读 → 用 → 回写** 循环：

```
轮开始：  读取 UserProfile（若无则冷启动 07）
推荐中：  用 taste_vector / price_anchor / dislikes 驱动召回(02)与精排(03)
轮结束：  根据本轮交互回写：
          · 用户确认下单某项 → 对应 taste 维度 +，frequent 更新
          · 用户说"太贵了"   → price_anchor 下调，记 too_expensive 反馈
          · 用户说"吃腻了X"  → X 的 novelty 惩罚，记 bored 反馈
          · 用户声明忌口     → 写入 dislikes（永久硬约束）
          · 用户尝试新品     → exploration_state.tried_new 追加
```

## 四、记忆的落地方式（在无外部存储前提下）

本 Skill 不依赖任何外部数据库，记忆通过两种方式实现，按运行环境择一：

1. **会话内记忆（默认）**：UserProfile 作为对话上下文的一部分，在多轮对话中由 Agent 维护与携带。
2. **跨会话记忆（可选增强）**：若运行环境支持长期记忆/用户变量（如部分 Agent 平台），把 UserProfile 序列化存入，下次会话读回。
3. **随时可重建**：即使记忆丢失，也能用 `order-list` 等接口**即时重建**画像，保证鲁棒性——这正是"冷数据可随时回放"的推荐系统设计。

## 五、隐私与边界

- 画像仅服务于点餐推荐，不收集与点餐无关的敏感信息。
- 画像内容对用户透明：用户可以问"你觉得我是什么口味"，决策官应如实复述画像要点。
- 用户可要求"忘掉某个偏好/重置画像"，应立即执行。
