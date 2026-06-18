# Blueprint Guide / 蓝图指南

## 1. 添加组件 / Add the component

1. 打开任意 Actor 蓝图（或在关卡里选中一个 Actor）。
2. `Add Component → Wake Word Component`。
3. 在 Details 面板里展开 `WakeWord` 分组配置（见下）。

## 2. 最小连线 / Minimal graph

```
Event BeginPlay ──▶ Wake Word Component ▸ Start Listening
                    （或勾选组件的 Auto Start On Begin Play，免连线）

Wake Word Component ▸ On Wake Word Detected
        │ Keyword (String)
        ▼
   你的逻辑（开始对话 / 播放反馈 / 点亮 UI ...）
```

## 3. 语言 / Language（中文 or 英文）

唤醒词语言在 **项目设置 → Plugins → Wake Word → `Language`** 里选：`Chinese`(kws-zh 拼音) 或
`English`(kws-en BPE)。组件的 `Model → Preset` 默认 **Auto**，自动跟随该设置。
要在一个 Actor 上中英文同时支持，就挂两个组件，一个 `Preset=Chinese`、一个 `Preset=English`。

The wake-word language is chosen in **Project Settings → Plugins → Wake Word → `Language`**.
The component's `Model → Preset` defaults to **Auto** and follows it.

## 4. 属性 / Properties

### WakeWord | 1 Model
- **Model → Preset** — `Auto`(跟随项目语言, 默认) / `Chinese`(kws-zh) / `English`(kws-en) / `Custom`。
  选 `Custom` 时才需要填子目录与文件名（高级折叠项）。

### WakeWord | 2 Tuning
- **Keywords Threshold** `0.05–0.5`（默认 `0.25`）— 全局检测阈值，越大越保守。
- **Keywords Score**（默认 `1.5`）— 关键词 boosting，越大越易命中。
- **Use Project Wake Words**（默认 `true`）— 用项目设置里的唤醒词（按 Language 自动分词）。
- **Num Threads**（高级，默认 `1`）— ONNX 推理线程数。
- **Custom Keywords (raw tokens)**（高级）— 运行时唤醒词行（每行一个，格式同 `keywords.txt`）。非空时**覆盖**一切。

### WakeWord | 3 Behavior
- **Auto Start On Begin Play**（默认 `false`）— BeginPlay 自动开始监听。
- **Retrigger Cooldown (s)** `0–5`（默认 `1.0`）— 命中后冷却，避免重复触发。

## 5. 事件 / Events

| 事件 / Event              | 参数 / Params              | 何时触发 / When |
|---------------------------|----------------------------|-----------------|
| `On Wake Word Detected`   | `Keyword: String`          | 检测到唤醒词    |
| `On Listening Started`    | —                          | 麦克风已开、模型就绪 |
| `On Error`                | `Code: Int, Message: String` | 出错（Code<0=客户端） |

## 6. 函数 / Functions

组件 / Component:
- `Start Listening` / `Stop Listening`
- `Is Listening` → bool
- `Is Ready` → bool — 调用 `Start Listening` 前可先检查（原生库已加载且模型文件齐全）。
- `Refresh Wake Words` — 改完项目设置/蓝图唤醒词后调用，立即生效（监听中会重建）。
- `Get Model Dir` → String — 解析后的模型目录（排查路径问题用）。
- `Set Custom Keywords (Keyword Lines: Array<String>, bRestart If Listening: bool)` — 运行时换词。

库 / Library (`UWakeWordLibrary`):
- `Get Language` / `Set Language (Language, bSave)` — 切换中文/英文。
- `Add / Remove / Clear / Get / Has Wake Word`、`Preview Tokens`、`Save Wake Words`。

## 7. 常见用法 / Recipes

**唤醒后切到对话，结束再恢复 / Wake → talk → resume**
```
On Wake Word Detected ─▶ Stop Listening ─▶ (启动你的对话组件)
对话结束回调          ─▶ Start Listening
```

**切到英文唤醒词 / Switch to English wake words**
```
Set Language (English, bSave=true) ─▶ Add Wake Word ("OPEN THE DOOR") ─▶ Wake Word Component ▸ Refresh Wake Words
```

**运行时切换一组唤醒词 (高级, 原始 token) / Swap keyword set at runtime (advanced, raw tokens)**
```
中文 / zh:  Make Array("n ǐ h ǎo sh uāng w ā @你好双蛙") ─▶ Set Custom Keywords (bRestart=true)
英文 / en:  Make Array("▁HE LL O ▁F RO G @HELLO FROG")   ─▶ Set Custom Keywords (bRestart=true)
```

## 8. 排错 / Troubleshooting

- **没反应** — 先 `Is Ready` 是否为 true；看 Output Log 里 `LogWakeWord` 行。`On Error` 的 Message 会说明缺哪个文件。
- **误唤醒多** — 调高 `Keywords Threshold` 到 0.35，或给该词加 `#0.30`。
- **漏唤醒** — 降低阈值，或给该词加 `:2.0`，并尽量用 4 音节以上的词。
- **麦克风打不开** — 检查系统默认录音设备与 Windows 麦克风权限；`On Error` 会报 code `-3`。
