# 短剧时间轴 Schema 与事件分类法

## 1. 单场景时间轴 Schema

多场景输出时，将多个场景对象放入 JSON 数组。

```json
{
  "scene_id": 12,
  "start": 32.1,
  "end": 38.5,
  "story_event": "identity_reveal",
  "emotion": {
    "type": "shock",
    "intensity": 0.92
  },
  "dialogue": {
    "importance": 0.98,
    "compress_pause": false
  },
  "editing": {
    "pace": "fast",
    "preferred_cut": "reaction",
    "shot_duration": 0.8,
    "transition": "black_flash"
  },
  "music": {
    "action": "mute_before_reveal"
  },
  "sfx": [
    { "name": "悬疑弦乐渐强", "offset": -0.7 },
    { "name": "咚咚垂音 心头震", "offset": 0 }
  ],
  "cliffhanger_score": 0.84
}
```

### 字段说明

| 字段 | 取值 | 说明 |
|---|---|---|
| `story_event` | 见下方事件分类 | 本场景的剧情事件类型 |
| `emotion.type` | `calm / tension / suspicion / conflict / shock / reversal / climax / sadness / fear / humor / romance` | 情绪类型 |
| `emotion.intensity` | 0.0 ~ 1.0 | 情绪强度 |
| `dialogue.importance` | 0.0 ~ 1.0 | 对白重要度（>0.8 视为重要对白） |
| `dialogue.compress_pause` | bool | false = 保留甚至加强戏剧性停顿 |
| `editing.pace` | `slow / normal / fast / very_fast` | 剪辑速度 |
| `editing.preferred_cut` | `reaction / dialogue / action / insert / cutaway` | 首选剪辑方式 |
| `editing.shot_duration` | 秒 | 建议镜头时长 |
| `editing.transition` | `white_flash / black_flash` | 项目约定：片尾收束用闪白，场景切换或悬念断点用闪黑 |
| `music.action` | `normal / duck / duck_hard / mute_before_reveal / silence` | 音乐动作 |
| `sfx[].name` | 音效库名称或待确认名称 | 见 `jianying-sfx-library.md` |
| `sfx[].offset` | 秒（相对台词/事件点） | 负值 = 事件前，0 = 事件瞬间 |
| `cliffhanger_score` | 0.0 ~ 1.0 | 结尾悬念强度 |

## 2. 剧情事件分类（story_event）

drama_rhythm 需要识别的全部节点类型：

- 对白信息点（dialogue_info）
- 情绪变化点（emotion_shift）
- 人物反应（reaction）
- 秘密揭露（secret_reveal）
- 冲突（conflict）
- 打脸（face_slap）
- 反转（reversal）
- 威胁（threat）
- 身份揭晓（identity_reveal）
- 误会（misunderstanding）
- 动作爆点（action_beat）

## 3. 情绪曲线类型（emotion.type）

判断标准：

| 情绪 | 特征 |
|---|---|
| `calm` | 正常叙事，无对抗 |
| `tension` | 压抑、试探、暗流 |
| `suspicion` | 怀疑、发现异常 |
| `conflict` | 争吵、对峙、正面冲突 |
| `shock` | 震惊、爆料、结果前置 |
| `reversal` | 身份反转、剧情反转 |
| `climax` | 高潮、爆发 |
| `sadness` | 情感低落、告别 |
| `fear` | 威胁、危险逼近 |
| `humor` | 喜剧、打脸 |
| `romance` | 浪漫、心动 |

## 4. 停顿分类（dialogue_editor 依据）

### 无意义停顿 → 自动缩短
呼吸、口癖、找词、没表演价值的停顿、镜头等待

### 戏剧性停顿 → 保留甚至加强
反转前、爆料前、人物震惊、威胁、死亡信息、身份揭晓、情感爆发

## 5. 音乐分段（music_director 依据）

| 剧情状态 | 音乐 |
|---|---|
| 普通叙事 | 弱音乐 |
| 怀疑 | tension pad |
| 发现异常 | pulse |
| 冲突 | music build |
| 反转前 | riser |
| 反转瞬间 | 静音 |
| 爆点 | impact |
| 爆点后 | climax music |

## 6. 音效绑定原则（sfx_director 依据）

音效不直接绑定“紧张”，要绑定剧情事件。示例：

```json
{
  "event": "secret_reveal",
  "emotion": "tension",
  "intensity": 0.91,
  "sfx_sequence": [
    { "type": "riser_short", "offset": -0.8 },
    { "type": "silence", "offset": -0.15 },
    { "type": "deep_hit", "offset": 0.0 }
  ]
}
```

## 7. 时间字段约定

`start` 与 `end` 使用秒，表示该段在交付时间轴上的起止位置；若要从原始素材中截取不同区间，执行适配层还需附带并映射对应的源素材时间范围。此 JSON 描述剪辑决策，不包含媒体文件路径，也不保证可直接传给某一版本的剪映自动化 API。
