# 自动降级私有建筑 — 说明文档

## Mod 解决什么问题？

Victoria 3 后期（~1880 年以后），AI 控制的私人投资会过度建造工厂，这些建筑招不到人、不赚钱，但玩家没法手动降级——私有等级没有 UI 入口。结果就是基础设施被撑爆，经济循环崩溃。

这个 Mod 做一件事：**定期检查全国建筑，把"没人干活也不赚钱"的私有等级自动降下来**。玩家仍然可以手动管理政府持有的等级，Mod 只碰玩家碰不到的那部分。

## 两层架构：玩家 + AI

Mod 有两套独立的降级逻辑，分别服务于玩家和 AI 国家：

```
玩家路径（可控制）
  JE 脉冲（每周/每月可选）
    → auto_downsize_check_and_execute
    → 两档严重程度 + 产权保护 + 冷却期 + 统计

AI 路径（全自动）
  on_half_yearly_pulse_country（每半年）
    → auto_downsize_ai_check
    → 仅严重档 + 更严阈值 + 产权保护 + 冷却期
```

| 维度 | 玩家 | AI 国家 |
|------|------|---------|
| 触发频率 | 每周或每月（按钮切换） | 每半年（固定） |
| UI 控制 | 日志条目 + 两个按钮 | 无 |
| 人手阈值 | < 50% | < 20% |
| 现金阈值 | < 25% | < 5% |
| 盈利阈值 | ≤ 0 | < -200 |
| 严重程度 | 严重 + 中度两档 | 仅严重档 |
| 冷却期 | 6 个月 | 6 个月 |
| 统计显示 | JE 中显示 | 无 |

AI 版故意设得更保守——AI 经济脆弱，只在建筑"彻底废了"时才出手，避免雪上加霜。

## 预防机制：AI 建造约束

除了降级已建成的建筑，Mod 还通过 `common/defines/` 覆盖了游戏 AI 的建造决策参数，从源头减少无效建造：

| Define | 原值 | 新值 | 效果 |
|--------|------|------|------|
| `PRODUCTION_BUILDING_NO_AVAILABLE_WORKFORCE_FACTOR` | 0.25 | **0** | 当地无人力时权重归零 → 禁止建造（总分乘数） |
| `PRODUCTION_BUILDING_STATE_NO_AVAILABLE_WORKFORCE_FACTOR` | 0.05 | **0** | 无人力州直接排除（总分乘数） |
| `PRODUCTION_BUILDING_LOW_EMPLOYMENT_THRESHOLD` | 0.8 | **0.9** | 雇佣率 < 90% 不扩建（有硬编码例外） |
| `PRODUCTION_BUILDING_AUTONOMOUS_INVESTMENT_BELOW_DESIRED_INFRASTRUCTURE_FACTOR_MULT` | 0.25 | **1.0** | 私有投资对基建的敏感度与政府持平 |
| `PRODUCTION_BUILDING_FREE_INFRASTRUCTURE_TARGET_WHEN_LACKING_WORKFORCE` | 5 | **10** | 缺人力时更早停止在低基建州建造 |

**局限性**：
- "上周招到人的建筑即使雇佣率低也可扩建"是引擎硬编码例外，defines 改不了
- 新建筑初始雇佣率 0%，不受 `LOW_EMPLOYMENT_THRESHOLD` 限制
- 基建的 MULT 是单项因子乘数而非总分，只能加大惩罚不能强制禁止

## 文件结构

```
v3-auto-downsize/
├── .metadata/
│   └── metadata.json                    # 启动器识别信息
├── common/
│   ├── defines/
│   │   └── auto_downsize_defines.txt    # AI 建造约束（预防）
│   ├── journal_entries/
│   │   └── auto_downsize_je.txt         # 玩家日志条目 + 脉冲
│   ├── scripted_buttons/
│   │   └── auto_downsize_buttons.txt    # 启用开关 + 频率切换
│   ├── scripted_effects/
│   │   └── auto_downsize_effects.txt    # ★ 核心：玩家 + AI 降级逻辑（治疗）
│   └── on_actions/
│       └── auto_downsize_on_actions.txt # AI 半年度钩子
└── localization/
    ├── english/auto_downsize_l_english.yml
    └── simp_chinese/auto_downsize_l_simp_chinese.yml
```

## 核心逻辑详解

`auto_downsize_effects.txt` 是整 mod 的大脑，玩家和 AI 的降级逻辑都在这个文件里。以下是玩家版的完整流程（AI 版是它的简化子集）。

### 流程总览

```
遍历全国每个建筑
  → ① 过滤：跳过不该碰的建筑
  → ② 条件：检查是否亏损缺人
  → ③ 分级：判断严重 / 中度
  → ④ 定量：根据等级决定降多少
  → ⑤ 保护：确保只降私有等级
  → ⑥ 执行 + 冷却 + 统计
```

### ① 豁免名单 — 这些建筑绝不降级

| 建筑组 | 例子 | 原因 |
|--------|------|------|
| `bg_government` | 大学、行政大楼 | 政府出资 |
| `bg_military` | 军营、海军基地 | 军事设施 |
| `bg_public_infrastructure` | 政府铁路 | 公共基建 |
| `bg_owner_buildings` 及其子组 | 金融区、庄园宅邸、公司总部 | 持有其他建筑的"壳"，误删会连锁 |
| `bg_trade` | 贸易中心 | 自动管理 |
| `bg_extraction` | 矿场、伐木场、渔场、油田、橡胶园 | 依赖地块资源禀赋，不应因短期亏损降级 |
| 自给建筑 | 自给农场/牧场 | 非私人投资 |
| `private_ownership_fraction = 0` | 完全政府持有的等级 | 玩家可手动操作，无需 mod 介入 |

### ② 触发条件 — 四项全满足才进入降级流程

| 条件 | 玩家阈值 | AI 阈值 | 含义 |
|------|---------|--------|------|
| 有私有产权 | `> 0` | `> 0` | 只处理有私人/公司股份的建筑 |
| 人手不足 | `< 50%` | `< 20%` | 岗位大面积空置 |
| 现金枯竭 | `< 25%` | `< 5%` | 建筑存款见底 |
| 持续亏损 | `≤ 0` | `< -200` | 不赚钱甚至严重亏本 |

### ③ 严重程度

| | 🔴 严重 | 🟡 中度（仅玩家） |
|---|---|---|
| 条件 | `occupancy < 0.25 AND cash < 0.1` | 满足触发条件但不满足严重 |
| 适用 | 玩家 + AI | 仅玩家 |

### ④ 降级量表

| 建筑等级 | 🔴 严重 | 🟡 中度（仅玩家） |
|----------|--------|-------------------|
| 1-10 级 | -1 | -1 |
| 11-25 级 | -3 | -2 |
| 26-50 级 | -5 | -3 |
| 51-100 级 | -8 | -5 |
| 100+ 级 | -12 | -8 |

### ⑤ 产权保护 — 确保不会误降政府等级

`add_building_level` 减少的是建筑的总等级，游戏引擎自行决定扣私有还是政府。为了确保政府等级纹丝不动，每个降级档位都有 `private_ownership_fraction` 最低门槛：

**门槛公式**：`降级量 ÷ 该档最低等级`

| 降级档 | 严重门槛 | 中度门槛 | 含义（以最低等级计算） |
|--------|---------|---------|----------------------|
| 100+ 级 | 12% | 8% | 100 级建筑至少要有 12 / 8 级私有 |
| 51-100 级 | 16% | 10% | 51 级建筑至少要有 8 / 5 级私有 |
| 26-50 级 | 20% | 12% | 26 级建筑至少要有 5 / 3 级私有 |
| 11-25 级 | 28% | 19% | 11 级建筑至少要有 3 / 2 级私有 |
| 1-10 级 | 50% (保守) | 50% (保守) | 低等级建筑只有私有过半才动 |

> **例子**：50 级纺织厂，30 级私有（60%）+ 20 级政府（40%），判定严重降 5 级。门槛 20%，实际 60% ≥ 20% → 通过。私有部分从 30 → 25，政府 20 级保持不变。

### ⑥ 冷却期与统计

- **冷却期**：每次降级后该建筑 6 个月内不再碰，防止反复抖动
- **统计**：`auto_downsize_buildings_affected` 变量累计被降级的建筑次数，显示在 JE 描述中

## 快速修改指南

每个文件里都有 `【可调参数】`、`【降级量表】`、`【冷却期】`、`【产权检查】` 标签，搜索即可直达。常见场景：

| 你想做什么 | 去哪改 |
|-----------|--------|
| 调降级力度 | `effects.txt` → 搜索 `【降级量表】` → 改数字 |
| 调触发灵敏度 | `effects.txt` → 搜索 `【可调参数】` → 改阈值 |
| 调冷却时间 | `effects.txt` → 搜索 `【冷却期】` → 改 `months = 6` |
| 调产权保护门槛 | `effects.txt` → 搜索 `【产权检查】` → 改 fraction 值 |
| 调 AI 触发条件 | `effects.txt` → AI 版块注释中有 `【可调参数】` |
| 默认启用 Mod | `je.txt` → `immediate` 块里加 `set_variable = auto_downsize_enabled` |
| 默认每周检查 | `je.txt` → `immediate` 块里加 `set_variable = auto_downsize_weekly` |
| 换个 JE 图标 | `je.txt` → 改 `icon = "..."` 的路径 |
| 隐藏 JE 但不停止功能 | `je.txt` → `should_be_pinned_by_default = no` |
| 免除更多建筑类型 | `effects.txt` → 在排除区加 `is_building_group = bg_xxx` |

## 安装与测试

```bash
cp -r v3-auto-downsize ~/Documents/Paradox\ Interactive/Victoria\ 3/mod/
```

1. 启动游戏 → 播放集 → 启用 "Auto Downsize Private Buildings"
2. 加载后期存档 → 打开日志面板（F8）→ 找到"私营建筑监管"
3. 点击启用按钮 → 快进 1-2 个月
4. 观察建筑等级变化，同时监控错误日志：

```bash
tail -f ~/Documents/Paradox\ Interactive/Victoria\ 3/logs/error.log | grep auto_downsize
```

## 已知风险

`add_building_level = { building = PREV level = -N }` 中的 `PREV` 需要引擎从建筑 scope 解析为建筑类型 key。游戏代码中有类似用法（`transfer_character = PREV`），但 `building` 参数是否支持有待实测。如果 error.log 报相关错误，需要改用硬编码建筑类型列表——逻辑正确但更繁琐。
