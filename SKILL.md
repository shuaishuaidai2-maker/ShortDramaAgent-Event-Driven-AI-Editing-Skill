---
name: short-drama-agent
description: 短剧剪辑全能总控 Skill（单 Skill 一体式）。用户提出任何短剧/微短剧/剧情短视频/竖屏剧的剪辑、制作、节奏调整、配乐、音效、字幕、悬念设计、成片质检、剪映落地需求（如"剪短剧""做微短剧""帮我剪辑短剧""短剧开头/结尾""短剧配乐音效""在剪映里做短剧"）时，只需命中本 Skill 一个，即可自动完成全部环节：hook 检测→情绪曲线→节奏控制→对白编辑→配乐→剪映音效→反应镜头→闪白/闪黑→悬念结尾→字幕→连续性→质检，并输出可交付的短剧时间轴 JSON 与剪映工程执行方案。本 Skill 内嵌完整 12 环节决策规则、剪映常用音效库、闪白/闪黑转场规则与 jianying-editor 落地指引，外部剪辑器自动化需要单独安装并配置。
---

# ShortDramaAgent: Event-Driven AI Editing Skill

短剧剪辑决策与后期规划 Skill

**本 Skill 为短剧剪辑提供统一的决策入口。** 触发后直接按下方 12 步流程跑完，交付短剧时间轴 JSON + 剪映工程执行方案。不要反问用户"用哪个功能"。

## 输入准备

- 剧本/分镜：直接用用户提供的文本。
- 原始视频：先读时间线/转写对白，建立事件清单。
- 素材缺失：明确告知缺什么，其余环节照常推进。

---

## 完整工作流（12 步，顺序执行）

### 第 1 步 · Hook（前 3 秒）
**开头必须抓人，禁止"天空→楼房→走路→对白"式开场。**
- 从全片找最强内容前置：强冲突/强情绪/悬念/身份反转/结果前置/危险/羞辱/抓奸/争吵/威胁/秘密。
- 结果前置标准格式：爆点台词 CUT → 字幕"三小时前" → 回到叙事。
- 前 3 秒无 hook → 必须重排；hook 强度 <0.8 继续找更强的。
- 输出：`candidate_hook`（source_time/content/hook_type/hook_strength）+ `recommended_open`（reorder/plan）。

### 第 2 步 · 情绪曲线
**把全片切成情绪段落并标强度。**
- 类型：calm / tension / suspicion / conflict / shock / reversal / climax / sadness / fear / humor / romance。
- 强度：0~0.3 铺垫，0.3~0.6 渐起，0.6~0.8 冲突，0.8~1.0 爆发/反转/高潮。
- 铁律：反转前必须有 tension/suspicion 铺垫；高潮必须全片最高点且之后有余韵。
- 输出：`emotion_curve`（start/end/type/intensity）+ `peaks`。

### 第 3 步 · 节奏（核心）
**节点驱动剪辑，禁止平均剪。** 识别 11 类节点：对白信息点/情绪变化点/人物反应/秘密揭露/冲突/打脸/反转/威胁/身份揭晓/误会/动作爆点。

| 节点 | 剪辑方式 |
|---|---|
| 对白信息点 | 台词结束立即 CUT |
| 秘密揭露 | 静音 0.25s + 台词 + IMPACT |
| 冲突 | 快速对切 0.6~1.0s |
| 反转 | riser → 静音 → 台词 → impact |
| 威胁 | 放慢 + 特写 + 低音 1.0~1.5s |
| 打脸 | 快切 0.3~0.5s + 音效点 |
| 身份揭晓 | 停顿保留 + 反应镜头 |

黄金节奏范例（"你为什么骗我"）：
```
女主："你为什么骗我？" CUT
男主眼神 0.5s
女主特写 0.7s
男主："因为……" 音乐骤停
0.25s 黑/静音
男主："你爸是我杀的。" IMPACT
```
- 每段节奏必须有松紧对比：平静段镜头长、爆发段镜头短。

### 第 4 步 · 对白编辑
**去废话、压缩无意义停顿、保留戏剧性停顿。**
- 无意义停顿（呼吸/口癖/找词/镜头等待）→ 压到 ≤0.15s 或删除。
- 戏剧性停顿（反转前/爆料前/人物震惊/威胁/死亡信息/身份揭晓/情感爆发）→ 保留 0.5~1.2s 甚至加强。
- 判据：停顿后内容是否改变人物命运/关系。改变→戏剧性；不改变→无意义。
- 输出：`original` / `compressed` / `pauses`（position/type/action/duration）。

### 第 5 步 · 配乐
**按情绪段落配乐 + 音乐让位给对白。**

| 剧情状态 | 音乐 |
|---|---|
| 普通叙事 | 弱音乐 |
| 怀疑 | tension pad |
| 发现异常 | pulse |
| 冲突 | music build |
| 反转前 | riser |
| 反转瞬间 | 静音（完全 mute 0.2~0.8s） |
| 爆点 | impact |
| 爆点后 | climax music |

Ducking 电平：无对白 -12dB；正常对白 -20dB；重要对白 -25dB；反转台词 mute。
- 输出：`music_track`（segment_id/start/end/emotion/action/base_level_db/ducking/fade_in/fade_out）。

### 第 6 步 · 音效（含剪映常用音效库 + 闪白/闪黑）
**音效绑定剧情事件，不绑定情绪；silence 也是音效事件。**

事件→序列模板：
- secret_reveal = riser_short(-0.8) → silence(-0.15) → deep_hit(0)
- identity_reveal = riser_medium(-1.0) → silence(-0.2) → cinematic_impact(0)
- threat = tension_low + heartbeat；face_slap = comedy_pop；shock = shock_hit
- reversal = reverse_riser → silence → bass_drop

**剪映常用音效库（优先使用，剪映内按名搜索）**：

| 功能 | 剪映音效名 |
|---|---|
| 冲击/撞击 | 哐撞击轻响、咚咚、咚咚垂音心头震、定音鼓咚咚咚、获取能源 |
| 紧张/悬疑 | 叮咚(紧张)、紧张恐惧感持续上升、沉重心跳、突然、突然加速、悬疑氛围嗡长音、悬疑弦乐渐强、闪回 |
| 转场 | "呼"的转场音效、极速上升转场、神圣的光感、风铃音效 |
| 喜剧/可爱 | 腰腰、疑惑、卡通跳跃Q弹啾飞综艺语言、啾、咕啾-可爱、综艺咯 |
| 动作 | 利刃出鞘!、匕首、摩托车机车点火拧油门 |
| 环境 | 人声嘈杂 |

**闪白/闪黑规则（用户指定，必须遵守）**：
- **闪白 → 结尾用**：整段/整集结尾、片尾 logo 前、情绪收束（0.2~0.5s 白场）。
- **闪黑 → 切场景时用**：场景切换、时空跳转、悬念断点（0.3~0.5s 黑场，配"呼"的转场音效）。
- 断集悬念可用闪黑 CUT TO BLACK，但整集真正结尾收束**必须用闪白**。
- JSON 标记：`"editing": { "transition": "black_flash" | "black_flash" }`。

- 输出：`sfx`（name/offset/volume_db），name 优先取剪映音效名。

### 第 7 步 · 反应镜头
**优先剪接收信息的人。**
- 爆信息：台词拆半，中间插听者反应特写 0.4~0.7s，信息后插震惊特写 0.6~0.9s。
- 判据：台词是否改变听者命运/关系/认知 → 是则必须插反应。
- 输出：`reaction_inserts`（at/who/shot/duration）+ `final_impact`。

### 第 8 步 · 悬念结尾
**每段结尾让人想点下一集。**
- 识别：门开/身份将揭晓/电话来/看到某东西/枪声/秘密说一半/人物出现/结果将公布。
- 标准动作：话说一半 → CUT TO BLACK（闪黑）。
- cliffhanger_score：0~0.3 无悬念；0.6~0.8 好钩子；0.8~1.0 最佳断集点。
- 输出：`candidates`（time/cliffhanger_type/content/cliffhanger_score/action）+ `best_break`。

### 第 9 步 · 字幕
**字幕参与节奏，不只转写。**
- 单句 ≤16 字；时长 = 台词 + 0.3~0.5s；爆点句整句高亮；转折词（但/其实/原来/竟然/不是）高亮；一屏最多 2 个高亮词。
- 输出：`subtitles`（start/end/text/highlight/style）。

### 第 10 步 · 连续性检查
**查人物/动作/对白穿帮。**
- 检查服装道具位置、动作衔接方向、对白前后矛盾。
- 严重度：high（必须修）/ medium（建议修）/ low（忽略）。
- 输出：`issues`（severity/type/time/description/fix）+ `summary`。

### 第 11 步 · 最终质检
**交付前查拖沓/空镜/节奏塌陷。**
- 拖沓：>2s 无信息镜头、5~10s 无变化 → 修。
- 空镜：>1.5s 纯空镜 → 删或插反应。
- 节奏塌陷：爆点后立刻泄气、反转无铺垫、结尾 score<0.5 → 打回。
- 任一 high 问题 → rework，回到对应步骤修正，pass 才交付。
- 输出：`qc_result`（pass/scores/failures/verdict）。

---

## 第 12 步 · 剪映自动化落地（可选外部集成）

本 Skill 负责生成剪辑决策和结构化时间轴；仓库不包含剪映自动化程序。若环境中另行安装了 `jianying-editor`，可通过适配层把本 Skill 的时间轴字段映射到该工具支持的 API。不要假定不同版本的 wrapper 参数完全一致，执行前先核对本机安装版本的接口。

### 配置外部工具路径

把 `JIANYING_EDITOR_SKILL_PATH` 指向本机 `jianying-editor` 安装目录。路径由使用者自行配置，不写入 Skill 文件或提交到仓库：

```bash
export JIANYING_EDITOR_SKILL_PATH="/path/to/jianying-editor"
```

Python 适配脚本可先检查配置和 wrapper 是否存在：

```python
import os
from pathlib import Path

skill_root = Path(os.environ["JIANYING_EDITOR_SKILL_PATH"]).expanduser()
wrapper_path = skill_root / "scripts" / "jy_wrapper.py"
if not wrapper_path.is_file():
    raise FileNotFoundError(f"jianying-editor wrapper not found: {wrapper_path}")
```

### 时间轴映射与交付检查

- 把 `scene_id`、时间范围、剪辑方式、音乐动作、音效事件和字幕映射为目标 wrapper 支持的调用。
- 在执行前补齐实际媒体文件路径，并检查时间单位、轨道名、音量和转场参数。
- 草稿保存后，检查视频片段、BGM、字幕、音效事件和转场位置；macOS 上的最终导出方式取决于剪映版本和本机环境。
- 需要 SRT 或 MP4 时，使用已安装工具所提供且经版本核对的导出流程。

## 最终交付格式

统一输出**短剧时间轴 JSON**（单场景对象；多场景时使用对象数组）：

```json
{
  "scene_id": 12,
  "start": 32.1,
  "end": 38.5,
  "story_event": "identity_reveal",
  "emotion": { "type": "shock", "intensity": 0.92 },
  "dialogue": { "importance": 0.98, "compress_pause": false },
  "editing": { "pace": "fast", "preferred_cut": "reaction", "shot_duration": 0.8, "transition": "black_flash" },
  "music": { "action": "mute_before_reveal" },
  "sfx": [
    { "name": "悬疑弦乐渐强", "offset": -0.7 },
    { "name": "咚咚垂音 心头震", "offset": 0 }
  ],
  "cliffhanger_score": 0.84
}
```

完整 schema 与分类法见 `references/timeline-schema.md`，节奏范例见 `references/editing-examples.md`，剪映音效库见 `references/jianying-sfx-library.md`。

## 使用铁律

- 触发即接管：不要反问"你想用哪个功能"，直接跑完整 12 步。
- AI 负责判断剧情事件/情绪强度/声音类型；具体 BGM/SFX 文件从剪映固定音效库规则选择，不自由发挥。
- 音效缺失时标注"待补"并登记，不自行生成替代。
- 成片交付前必须过第 11 步质检，pass 才交付。
- **闪白结尾用、闪黑切场景用**——这是硬规则，不可颠倒。
