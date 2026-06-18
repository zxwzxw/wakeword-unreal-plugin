# Changelog — WakeWord

本插件遵循语义化版本。日期为 ISO 8601。
This plugin follows semantic versioning. Dates are ISO 8601.

---

## 1.1.0 — 2026-06-14

### 新增 / Added

- **英文唤醒词 / English wake words** — 内置 gigaspeech 3.3M 英文 KWS 模型
  (`Resources/kws-en`)。直接填英文原文（如 `OPEN THE DOOR`、`HELLO FROG`）即可，识别率与中文相当。

  New bundled gigaspeech 3.3M English KWS model. Type English phrases directly; accuracy on par
  with the Chinese model.

- **引擎内英文分词器 / In-engine English tokenizer** — 用与 sentencepiece 一致的 Viterbi
  分词（unigram，读取派生的 `kws-en/bpe-scores.txt`），**逐字复现**模型训练时的分词，
  运行时无需 Python、无需改文件。

  A sentencepiece-equivalent Viterbi tokenizer reproduces the gigaspeech model's tokenization
  exactly at runtime — no Python, no file editing.

- **语言切换 / Language switch** — 项目设置 → Plugins → Wake Word → **`Language`**（Chinese / English）
  一处切换模型与分词方式。蓝图新增 `Get Language` / `Set Language`。

  New `Language` setting (and Blueprint `Get/Set Language`) selects the model + tokenizer.

- **模型预设 / Model preset** — 组件 `Model → Preset`：`Auto`（跟随项目语言，默认）/ `Chinese` /
  `English` / `Custom`。一个 Actor 挂两个组件（`Preset=Chinese` + `Preset=English`）即可中英同时常驻。

  New component `Model → Preset` (Auto / Chinese / English / Custom). Two components = both
  languages at once.

- **English BPE offline tool** — `Tools/text2token.py` 新增 `--lang en --bpe-model`（sentencepiece），
  可离线生成/校验英文 token。New `--lang en` mode in `text2token.py`.

- **UE 5.7 支持 / UE 5.7 support** — 源码同时支持 5.6 与 5.7（5.7 经 `RunUAT BuildPlugin` 隔离构建验证）。

### 变更 / Changed

- 所有 UPROPERTY/UFUNCTION 显示名统一为**英文**，并补全中英双语 `ToolTip`。
  All editor display names are English now, each with a bilingual tooltip.

- 切换 `Language` 时，若唤醒词列表仍是出厂默认（未自定义），自动替换为该语言的默认词，
  避免留下与新模型不匹配的旧语言词。**自定义过的列表绝不会被改动。**

  Switching `Language` swaps the factory-default list to the new language's defaults; a
  customized list is never touched.

### 升级提示 / Upgrade notes

- 想用英文：项目设置 → Wake Word → `Language = English`，填英文唤醒词即可。
- 中英文同时：一个 Actor 挂两个 `Wake Word Component`，分别设 `Preset=Chinese` / `Preset=English`，各绑各的事件。
- 第二种语言的组件请**不要**勾 `Use Project Wake Words`（项目列表是单语言的）；改用该组件的
  `Custom Keywords (raw tokens)` 或内置 `keywords.txt`。

---

## 1.0.0 — 2026-06-13

- 首个发布版本：UE5.6 离线中文唤醒词检测（sherpa-onnx KWS + wenetspeech 3.3M），常驻麦克风、
  低延迟、零每次调用费用；引擎内拼音分词、项目设置/蓝图管理唤醒词、与 RealtimeVoice 对话管线联动。
  Initial release: offline Chinese keyword spotting for UE5.6.
