---
name: video-actor-organizer
description: 演员分类整理。解析 NFO 文件提取演员名，按拼音首字母/假名分类，同时自动识别VR视频。支持日文片假名演员的中文映射记忆系统。当用户提到"演员分类"、"按演员整理"、"actor classify"时触发。
---

# 演员分类整理 Skill

## 功能

1. 扫描视频文件夹，获取第一个 NFO 文件
2. 解析 NFO 提取演员名、标题和视频分辨率
3. **中文映射记忆系统**：
   - 自动识别包含日文假名的演员名
   - 查询记忆文件 `memory/actor_mappings.toml` 中的映射关系
   - 已映射：按中文映射名的拼音首字母分类，目录格式为 `中文名_原名`
   - 未映射：跳过该演员的视频，记录到未映射列表返回给 Agent
4. 按演员名分类：
   - 拼音首字母（A-Z）- 基于中文映射名或原名
   - 假名首字母（A-Z）
   - 日文名（无映射时）→ 暂不分类，等待映射
   - 未知演员 → 分类 `99`
5. 自动识别 VR 视频：
   - 宽度 > 阈值（默认2000像素）且非 16:9 宽高比的视为 VR 视频
   - VR 视频生成目标路径：`{target-root}/BIG/{category}/{首字}/{演员名}/{标题}/`
   - 普通视频生成目标路径：`{target-root}/{category}/{首字}/{演员名}/{标题}/`

## 扫描目录与目标目录分离

支持独立设置扫描目录（`--root`）和目标根目录（`--target-root`）：
- `--root`：扫描视频文件的目录
- `--target-root`：移动目标根目录（可选，默认与 `--root` 相同）

**使用场景**：扫描 A 文件夹，但以 JAV 为根目录整理，避免出现 `A/A/` 嵌套结构：

```bash
.venv/bin/python scripts/plan_actor_classify.py \
  --root "/mnt/nfs/jellyfin/JAV/A" \
  --target-root "/mnt/nfs/jellyfin/JAV" \
  ...
```

这样视频会整理到 `/mnt/nfs/jellyfin/JAV/A/庵/庵姬花_庵ひめか/...` 而不是 `/mnt/nfs/jellyfin/JAV/A/A/庵/...`

## 特殊处理

FC2/PPV 视频如果没有演员信息，自动添加 `FC2-PPV`, `FC2`, `PPV` 作为演员标签。

## 中文映射记忆系统

### 记忆文件

映射关系存储在 `memory/actor_mappings.toml`：

```toml
[actor_mappings]
"さつき芽衣" = "月芽衣"
"あおい空" = "青空"
"ゆい花" = "由衣花"
```

### 工作流程

1. **扫描阶段**：
   - 检测到片假名演员 `さつき芽衣`
   - 查询映射表 → 找到 `月芽衣`
   - 生成目标路径：`Y/月/月芽衣_さつき芽衣/`

2. **未映射处理**：
   - 检测到片假名演员 `新垣あい`
   - 查询映射表 → 未找到
   - **不生成**该视频的移动计划
   - 记录到未映射列表

3. **Agent 处理**：
   - 脚本输出 `UNMAPPED_ACTORS_START/END` 标记的未映射列表
   - Agent 读取列表，为每个演员生成合适的中文映射
   - 更新 `memory/actor_mappings.toml`
   - 用户可手动编辑映射名以使用喜欢的名称

4. **下次运行**：
   - 重新扫描时，已映射的演员正常分类
   - 目录格式：`中文映射名_原名`

### 映射命名示例

| 原名 | 中文映射 | 目标目录 |
|------|----------|----------|
| さつき芽衣 | 月芽衣 | `Y/月/月芽衣_さつき芽衣/` |
| あおい空 | 青空 | `Q/青/青空_あおい空/` |
| 未映射演员 | - | 暂不移动，等待映射 |

## 操作模式

三种模式：计划/执行/计划+执行。

## 调用方式

```bash
# 仅计划（扫描目录与目标目录相同）
.venv/bin/python active_skills/video-actor-organizer/scripts/plan_actor_classify.py \
  --root "/path/to/videos" \
  --unknown-category "99" \
  --japanese-category "0" \
  --title-max-length 20 \
  --width-threshold 2000 \
  --big-dir "BIG" \
  --output "plans/"

# 仅计划（扫描目录与目标目录分离，推荐）
.venv/bin/python active_skills/video-actor-organizer/scripts/plan_actor_classify.py \
  --root "/mnt/nfs/jellyfin/JAV/A" \
  --target-root "/mnt/nfs/jellyfin/JAV" \
  --unknown-category "99" \
  --japanese-category "0" \
  --title-max-length 20 \
  --width-threshold 2000 \
  --big-dir "BIG" \
  --output "plans/"

# 仅执行
.venv/bin/python active_skills/video-actor-organizer/scripts/execute_plan.py \
  --plan "plans/actor_20260220_150000.json"
```

## 核心组件

### classifier.py

自包含演员名分类逻辑：
- 依赖：`pypinyin`, `pykakasi`
- 功能：
  - 检测日文假名 `has_kana()`
  - 查询映射表 `get_mapping()`
  - 检查是否需要映射 `needs_mapping()`
  - 支持分类类型 `chinese_mapped`（已映射的中文名）

### plan_actor_classify.py

扫描和计划生成：
- 检测未映射的片假名演员
- 跳过未映射演员的计划生成
- 输出未映射列表供 Agent 处理
- 使用 `display_name`（中文名_原名）作为目录名

### execute_plan.py

执行移动计划。

### `execute_plan.py`（执行前强制步骤）

在调用执行脚本前，先清理 `plans/` 中超过 10 天的旧计划文件：

- 只处理 `plans/` 下普通文件（不递归目录）
- 按文件修改时间判断，`mtime > 10 天` 才删除
- 禁止使用 `rm`，统一使用 Python（`pathlib` / `os`）删除
- 清理后输出摘要：`删除数量 + 剩余数量`，再继续执行 `execute_plan.py`

## JSON 格式

```json
[
  {
    "func": "actor_classify",
    "action": "MOVE",
    "source": "/path/to/videos/SSNI-123",
    "destination": "/path/to/videos/Y/月/月芽衣_さつき芽衣/某个标题",
    "size_mb": 2048.0,
    "actor": "月芽衣_さつき芽衣",
    "original_actor": "さつき芽衣",
    "category": "Y",
    "title": "某个标题",
    "is_vr": false,
    "width": 1920,
    "height": 1080,
    "created_at": "2026-02-20T15:00:00"
  }
]
```

## 技能调用结论总结相关

在调用技能后，只需将结论稍微输出一下，不要做额外的检查核对等事项，py脚本输出的信息经过一轮整理已经足够清晰，不要做多余的浪费token的扫描等工作，不要额外再写一个脚本来做对json计划文件的再归纳。

## Agent 处理未映射演员的流程

1. 运行 `plan_actor_classify.py` 生成计划
2. 检查输出中是否有 `UNMAPPED_ACTORS_START` 标记
3. 如果有未映射演员：
   - 为每个片假名演员生成合适的中文映射
   - 更新 `memory/actor_mappings.toml`
   - 告知用户已更新的映射
   - 建议用户：可以编辑记忆文件修改映射名
4. 如果没有未映射演员：
   - 正常展示计划并询问是否执行


## 其他注意

- `--include-organized` 默认关闭。只有在用户明确说“无视已整理目录/强制重做”时才允许开启。
- 开启 `--include-organized` 后会扫描 A-Z/已整理目录，但如果目标路径与源路径相同则不会生成计划。
- 生成的计划路径，不能放在skill的目录中，要放在agent运行的目录中，保持skill目录清洁
