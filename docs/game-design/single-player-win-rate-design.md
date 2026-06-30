# 单机欧洲传统游戏：程序端胜率设计指南

本文档说明单机版欧洲传统游戏中，从程序架构角度如何设计、实现与验证「胜率」。

## 1. 先定义：单机里的「胜率」指什么

在单机游戏中，胜率很少等同于「真实数学概率」，通常需要明确统计的是哪一种：

| 类型 | 含义 | 典型场景 |
|------|------|----------|
| **客观胜率** | 给定局面/策略，长期赢的比例 | 残局、固定发牌、无 AI |
| **对 AI 胜率** | 玩家相对电脑对手的赢率 | 扑克、西洋双陆、跳棋等 |
| **通关/任务胜率** | 完成关卡或目标的比例 | 带剧情、关卡、Roguelike |

程序上必须先定义：**统计的是哪一种胜率**，否则后续配置、埋点、平衡都会混乱。

## 2. 推荐架构：分层设计

```
┌─────────────────────────────────────────┐
│  Experience Layer（体验层）              │
│  目标胜率、难度曲线、DDA、新手保护        │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Opponent / RNG Layer（对手与随机层）    │
│  AI 强度、发牌偏差、噪声、失误率          │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Rules Engine（规则引擎）                │
│  纯规则、无「帮玩家/坑玩家」              │
└─────────────────────────────────────────┘
```

**核心原则：规则引擎保持公平、可复现；胜率调节放在上层，不要写死在规则里。**

## 3. 四种常用实现手段

### 3.1 配置驱动的目标胜率（最常用）

用数据表定义各难度下的目标区间，而不是在代码里写死：

```json
{
  "difficulty": "normal",
  "target_win_rate": { "min": 0.45, "max": 0.55 },
  "ai": {
    "search_depth": 4,
    "mistake_probability": 0.12,
    "bluff_aggression": 0.6
  },
  "rng": {
    "seed_mode": "per_match",
    "deal_fairness": "uniform"
  }
}
```

程序职责：

- **Simulation**：用 bot 或蒙特卡洛跑 N 局，测当前配置的实测胜率
- **Tuning**：调 `mistake_probability`、`search_depth` 等，逼近目标区间
- **Runtime**：只读配置，不在对局中途改规则

### 3.2 AI 强度作为胜率杠杆（对战类）

对棋类、牌类 AI，胜率主要由以下因素决定：

- 搜索深度 / 思考时间
- 评估函数权重
- **故意失误率**（`mistake_probability`）
- 信息不完美时的保守/激进程度

```python
def choose_move(state, ai_config):
    best = engine.search(state, depth=ai_config.depth)
    if random() < ai_config.mistake_probability:
        return random_suboptimal_move(state)
    return best
```

优点：玩家容易接受（「对手变强/变弱」），比直接改发牌更不容易觉得作弊。

### 3.3 随机层设计（发牌、骰子、事件）

- **真随机**：`seed` 可复现，分布均匀（适合竞技、复盘）
- **约束随机**：保证「N 局内大数定律接近目标」（如保底好牌，慎用）
- **分桶 RNG**：按难度从不同分布采样

```python
def deal_hand(difficulty, rng):
    if difficulty == "tutorial":
        return rng.sample_from("beginner_friendly_deck")
    return rng.uniform_shuffle(full_deck)
```

注意：**明显偏袒玩家的发牌**短期爽，长期容易被识破；更稳的做法是 AI 弱一点 + 真随机。

### 3.4 动态难度（DDA，可选）

根据近期表现微调，而不是每局硬改结果：

```python
def adjust_difficulty(player_stats, base_config):
    recent_wr = player_stats.wins_last_20 / 20
    if recent_wr < 0.35:
        return base_config.with_delta(mistake_probability=+0.05)
    if recent_wr > 0.65:
        return base_config.with_delta(mistake_probability=-0.05)
    return base_config
```

要点：

- 用**滑动窗口**（如最近 20 局），避免单局波动
- 调节幅度小、有上下限
- 高级玩家可关 DDA，避免「系统在控分」的观感

## 4. 核心数据模型

建议至少持久化以下字段，方便平衡和排查：

```typescript
interface MatchRecord {
  matchId: string;
  seed: number;
  difficulty: string;
  playerId: string;
  result: "win" | "loss" | "draw";
  turns: number;
  playerSkillEstimate?: number;  // 可选 ELO/TrueSkill
  aiConfigSnapshot: object;      // 当局 AI 配置快照
  timestamp: number;
}

interface PlayerStats {
  totalMatches: number;
  wins: number;
  winRateByDifficulty: Record<string, number>;
  recentWindow: MatchRecord[];   // 最近 N 局
}
```

胜率计算：

```typescript
function winRate(matches: MatchRecord[], filter?: Filter): number {
  const filtered = applyFilter(matches, filter);
  if (filtered.length === 0) return 0;
  return filtered.filter(m => m.result === "win").length / filtered.length;
}
```

**分层统计**：按难度、关卡、版本分开算，不要只用一个全局胜率。

## 5. 平衡与验证流水线（程序侧必做）

```
规则实现 → 单元测试（合法走法）
    → Bot 对战模拟（1万+ 局）
    → 实测胜率 vs 目标区间
    → 调参 → 回归测试
    → 上线后埋点监控
```

### 蒙特卡洛示例思路

```python
def estimate_win_rate(player_policy, ai_config, n=10000):
    wins = 0
    for i in range(n):
        state = new_game(seed=i)
        while not state.is_terminal():
            if state.current_player == HUMAN:
                move = player_policy(state)
            else:
                move = ai_move(state, ai_config)
            state = state.apply(move)
        if state.winner == HUMAN:
            wins += 1
    return wins / n
```

用不同策略的 bot（新手型 / 最优型）测区间，比只测「随机玩家」更有意义。

## 6. 难度曲线怎么定（体验目标）

常见目标区间（可按游戏类型调整）：

| 玩家类型 | 目标胜率 | 设计意图 |
|----------|----------|----------|
| 新手教程 | 70%–85% | 建立信心 |
| 普通 | 45%–55% | 有挑战但不挫败 |
| 困难 | 25%–40% | 硬核 |
| 高手 / 挑战 | 15%–30% | 追求极限 |

程序上做成**曲线**而不是单点：

```python
def target_win_rate(level, player_matches_played):
    # 新手前 10 局略高，之后回落到难度目标
    base = DIFFICULTY_TABLE[level].target
    if player_matches_played < 10:
        return min(0.75, base + 0.15)
    return base
```

## 7. 欧洲传统游戏里要特别注意的点

- **完美信息棋类**（国际象棋等）：胜率几乎完全由 AI 强度决定；规则层不要动手脚。
- **不完全信息牌类**（Skat、Belote 等）：AI 质量 + 信息估计误差是主杠杆；发牌建议保持均匀。
- **运气+技巧**（西洋双陆、部分牌戏）：长期胜率靠 AI；单局波动大，UI 上要传达「这一局运气差」而不是「系统在搞你」。
- **单人纸牌**（接龙类）：通常是**关卡通过率**，用关卡设计 + 少量洗牌约束，而不是 PvP 式胜率。

## 8. 建议的项目模块划分

```
game/
  rules/           # 纯规则，无胜率逻辑
  ai/              # 对手，读 ai_config
  rng/             # 可复现随机
  balance/
    difficulty.json
    simulator.py   # 离线测胜率
    tuner.py       # 可选：自动搜参
  stats/
    match_logger.py
    win_rate.py
  dda/             # 可选动态难度
```

## 9. 实践建议（简短）

1. **规则与胜率分离**：规则可测、可复现；胜率在 AI / 配置 / DDA。
2. **用配置表 + 模拟器**，不要靠手感改常量。
3. **分层统计**：难度、版本、关卡分开看。
4. **优先调 AI**，慎用「帮玩家发牌」。
5. **每局存 seed 和配置快照**，方便复现「这局为什么这么难」。

## 10. 后续可扩展内容

若已确定具体游戏类型，可在此基础上补充：

- 具体游戏（国际象棋 / 接龙 / 某种扑克等）
- 是否有 AI 对手，还是纯解谜/闯关
- 目标受众（休闲 vs 竞技）
- 技术栈（Unity、Web、后端等）

对应的模块接口与配置表结构可单独成文。
