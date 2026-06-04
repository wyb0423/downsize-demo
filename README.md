# 自动降级私有建筑 Mod — 小白也能看懂的说明文档

## 这个 Mod 是干嘛的？

Victoria 3 游戏后期（大概 1880 年以后），AI 控制的私人投资（资本家、贵族）会疯狂建造工厂、矿山等建筑。但这些建筑经常招不到足够的人手，也不赚钱，结果就是：
- 基础设施被撑爆（每级建筑都吃基建容量）
- 经济崩盘（大量空置建筑拖累整个经济）

**更关键的是**：这些私有建筑玩家没法手动降级（UI 不允许），只能眼睁睁看着经济崩。

这个 Mod 做的是：**定期自动检查全国私有建筑，把那些"没人干活也不赚钱"的建筑等级逐步降下来**。

## 文件说明（从简单到复杂）

```
v3-auto-downsize/
├── .metadata/
│   └── metadata.json               ← ① 启动器识别信息（可以不看）
├── common/
│   ├── journal_entries/
│   │   └── auto_downsize_je.txt    ← ② 游戏内"日志"面板的条目
│   ├── scripted_buttons/
│   │   └── auto_downsize_buttons.txt ← ③ 日志条目上的交互按钮
│   └── scripted_effects/
│       └── auto_downsize_effects.txt ← ④ ★核心★ 降级检查逻辑
└── localization/
    ├── english/...                  ← ⑤ 英文文本
    └── simp_chinese/...             ← ⑥ 中文文本
```

### ① metadata.json — Mod 身份证

游戏启动器读取这个文件来知道"这个 Mod 叫什么、什么版本"。

| 字段 | 含义 | 需要改吗？ |
|------|------|-----------|
| `name` | Mod 显示名称 | 随便改 |
| `id` | 唯一标识符 | 不要改，改了可能导致存档不兼容 |
| `version` | 版本号 | 每次大改后递增 |
| `supported_game_version` | 支持的游戏版本 | 游戏大版本更新时可能需要改 |
| `tags` | 标签 | 不影响功能 |

### ② auto_downsize_je.txt — 游戏内的"日志条目"

**它在游戏中长什么样？** 打开"日志"面板（快捷键 F8），你会看到一条叫"私营建筑监管"的条目，下面有两个按钮。

逐行解释：
```
je_auto_downsize = {              ← 条目的内部名字（代码用）
    icon = "..."                  ← 左侧显示的图标
    group = je_group_internal_affairs  ← 在日志面板的分类（内政类）
    is_shown_when_inactive = {    ← 即使条目未激活也显示给玩家看
        is_player = yes
    }
    possible = { always = yes }   ← 始终可以激活
    immediate = { ... }           ← 游戏加载/条目激活时执行一次
    scripted_button = xxx         ← 挂载按钮（见③）
    on_weekly_pulse = { ... }     ← 每周触发一次
    on_monthly_pulse = { ... }    ← 每月触发一次
    weight = 100                  ← 在日志面板中的排序权重
    should_be_pinned_by_default = yes  ← 默认置顶
}
```

**工作流程**（关键！）：
```
每周/每月触发
    ↓
检查是否启用了 Mod？（auto_downsize_enabled 变量存在吗？）
    ↓ 是
检查频率是每周还是每月？（auto_downsize_weekly 变量存在吗？）
    ↓ 匹配
运行 auto_downsize_check_and_execute（④中的核心逻辑）
```

**如果你想修改**：
- 想让 Mod 只在特定国家显示？把 `is_player = yes` 改成国家条件
- 想默认启用？在 `immediate` 里加 `set_variable = auto_downsize_enabled`
- 不想看到这个条目？删掉 `is_shown_when_inactive` 块

### ③ auto_downsize_buttons.txt — 交互按钮

每个按钮的结构：
```
按钮名 = {
    name = "本地化key"       ← 按钮上显示的文字（在⑤⑥中定义）
    desc = "本地化key"       ← 鼠标悬停时的提示文字
    visible = { ... }        ← 什么时候显示这个按钮
    selected = { ... }       ← 什么时候按钮高亮（表示"已选中"状态）
    possible = { ... }       ← 什么时候可以点击（灰色=不可点击）
    effect = { ... }         ← 点击后发生什么
}
```

**按钮 1 — 启用/禁用开关**：
- 点击一次 → 设置 `auto_downsize_enabled` 变量 → Mod 开始工作
- 再点一次 → 删除 `auto_downsize_enabled` 变量 → Mod 停止
- 按钮高亮状态反映当前是否启用

**按钮 2 — 每周/每月频率切换**：
- 默认是"每月检查"（更省性能）
- 点击切换为"每周检查"（响应更快）
- 只有在启用了 Mod 时才能点击（`possible` 条件检查 `auto_downsize_enabled`）
- 按钮高亮状态反映当前是每周还是每月

**如果你想修改**：
- 想增加冷却时间防误触？加 `cooldown = { days = 30 }`
- AI 国家也想自动启用？改 `ai_chance = { base = 100 }`

### ④ auto_downsize_effects.txt — ★核心逻辑★（最重要！）

这是整个 Mod 的大脑。被 JE 的脉冲调用后，它会：

```
遍历全国的每一个建筑
    ↓
① 先过滤掉不该碰的建筑
    ↓
② 检查是否符合"需要降级"的条件
    ↓
③ 判断严重程度（严重 vs 中度）
    ↓
④ 根据建筑等级计算降多少级
    ↓
⑤ 执行降级 + 设冷却期 + 更新统计
```

**① 过滤逻辑 — 哪些建筑会被跳过？**

| 被跳过的建筑组 | 游戏中的例子 | 跳过原因 |
|---------------|-------------|---------|
| `bg_government` | 大学、政府大楼、艺术院 | 政府出钱的，不是私人投资 |
| `bg_military` | 军营、海军基地 | 军事建筑 |
| `bg_public_infrastructure` | 政府铁路 | 公共基建 |
| 自给建筑 | 自给农场、自给牧场 | 农民自给自足，不是资本家建的 |
| `bg_owner_buildings` | 金融区、庄园宅邸、公司总部 | 这些是"持有其他建筑"的建筑，删了会有连锁反应 |
| `bg_trade` | 贸易中心 | 贸易自动管理 |

**② 触发条件 — 同时满足下面三条才会被降级：**

| 条件 | 代码 | 阈值 | 含义 |
|------|------|------|------|
| 人手不足 | `occupancy < 0.5` | 低于 50% | 只有不到一半的岗位有人干活 |
| 现金见底 | `cash_reserves_ratio < 0.25` | 低于 25% | 建筑的"存款"快花光了 |
| 不赚钱 | `weekly_profit <= 0` | 零或负数 | 每周收支打平或亏本 |

**③ 严重程度分级：**

| 等级 | 条件 | 代码 |
|------|------|------|
| 🔴 严重 | 人手 < 25% 且 现金 < 10% | `occupancy < 0.25 AND cash_reserves_ratio < 0.1` |
| 🟡 中度 | 人手 < 50% 且 现金 < 25%（但不满足严重条件） | `occupancy < 0.5 AND cash_reserves_ratio < 0.25` |

**④ 降级数量表（核心！你最容易修改的地方）：**

| 建筑等级 | 🔴 严重降级量 | 🟡 中度降级量 |
|----------|-------------|-------------|
| 1-10 级  | -1 | -1 |
| 11-25 级 | -3 | -2 |
| 26-50 级 | -5 | -3 |
| 51-100 级 | -8 | -5 |
| 100+ 级 | -12 | -8 |

> **修改指南**：如果你觉得降得太猛了，把负数改小（比如 -12 改成 -6）。
> 如果你觉得降得太慢，把负数改大。每个等级段的数字都可以独立调整。

**⑤ 冷却期**：
- 每次降级后，该建筑 6 个月内不会再被降级
- 代码：`set_variable = { name = auto_downsize_cooldown months = 6 }`
- 想改冷却时间？把 `months = 6` 改成你想要的月数

**⑥ 统计追踪**：
- 每次降级成功，`auto_downsize_buildings_affected` 变量 +1
- 这个数字会显示在日志条目描述中（"已处理的建筑数: X"）

### ⑤⑥ localization — 本地化文本

YAML 格式，`key:数字 "文本"` 的格式。

颜色代码：
- `#bold ...#!` — 粗体
- `#variable ...#!` — 变量色（游戏主色调）
- `\n\n` — 换行

变量显示语法：
```
[SCOPE.sCountry('root').MakeScope.Var('变量名').GetValue|0]
```
这会从国家 scope 读取变量值并显示出来。`|0` 表示如果变量不存在显示 0。

## 常见修改场景

### 场景 1："我觉得降级太激进了"
打开 `common/scripted_effects/auto_downsize_effects.txt`，找到降级数量表，把数字改小。

### 场景 2："我想改检查频率的默认值"
打开 `common/journal_entries/auto_downsize_je.txt`，在 `immediate = { ROOT = { ... } }` 里加一行：
```
set_variable = auto_downsize_weekly   ← 默认改为每周
```

### 场景 3："我想把触发阈值调高/调低"
打开 `common/scripted_effects/auto_downsize_effects.txt`：
- 改 `occupancy < 0.5` 的 0.5 → 0.3 表示"人手低于30%才触发"
- 改 `cash_reserves_ratio < 0.25` 的 0.25 → 0.1 表示"现金低于10%才触发"

### 场景 4："我不想看到这个日志条目但 Mod 继续运行"
把 `should_be_pinned_by_default = yes` 改成 `should_be_pinned_by_default = no`

### 场景 5："MOD 报错了，error.log 在哪？"
```
~/Documents/Paradox Interactive/Victoria 3/logs/error.log
```
用文本编辑器打开，搜索 `auto_downsize` 关键词。

## 技术风险说明

**最大的不确定项**：`add_building_level = { building = PREV level = -N }`
- `PREV` 在这里的意思是"前一个 scope（即被遍历到的建筑）"
- 理论上应该能工作（游戏其他地方有类似用法）
- 如果 error.log 报这个错，说明 PREV 不能在这里用
- 届时需要改成硬编码建筑类型列表的方式（略麻烦但一定行）
