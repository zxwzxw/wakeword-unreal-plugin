# WakeWord — On-Device Wake-Word / Keyword Spotting for UE 5.6 / 5.7

离线、低延迟、常驻麦克风的**中文 + 英文**唤醒词检测插件。基于 [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (k2-fsa, Apache-2.0) 的 Keyword Spotting 引擎。

Offline, low-latency, always-on keyword spotting for Unreal Engine 5.6 / 5.7 — **Chinese and English** — powered by the
[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) KWS engine.

- **离线 / Offline** — 100% on-device. 无网络、无每次调用费用、无隐私外泄。No network, no per-call cost.
- **低延迟 / Low latency** — 单次推理 < 10ms，桌面端 CPU 占用可忽略。
- **中英双模型 / Chinese + English** — 内置中文 (wenetspeech) 与英文 (gigaspeech) 两个模型，一个开关切换。
- **零训练自定义 / Zero-shot custom keywords** — 填中文或英文原文即可，引擎内自动分词，无需重新训练。
- **蓝图友好 / Blueprint-friendly** — 一个组件 + 一个事件即可接入。
- **数字人零阻塞 / Render-safe** — 音频在后台线程处理，命中后回到 GameThread 广播。

> 平台 / Platform: **Win64**. 引擎 / Engine: **UE 5.6 / 5.7**.
> 模型 / Models: wenetspeech 3.3M (中文/Chinese, `kws-zh`) · gigaspeech 3.3M (英文/English, `kws-en`).

---

## 快速开始 / Quick start

1. **启用插件 / Enable the plugin** — 复制 `WakeWord/` 到工程 `Plugins/`，在 `.uproject` 中启用，或在编辑器 `Edit → Plugins` 里勾选。重启编辑器让其编译。
2. **选语言 / Pick the language** — 项目设置 → Plugins → **Wake Word** → `Language` 选 **Chinese** 或 **English**。
3. **挂组件 / Add the component** — 给任意 Actor 添加 `Wake Word Component`（`Model → Preset` 默认 **Auto**，跟随上面的语言）。
4. **（可选）填唤醒词 / (optional) add wake words** — 在同一面板的 `Wake Words` 里加几条，填中文或英文原文，`Tokens` 自动显示。
5. **开始监听 / Start listening** — 勾选组件的 `Auto Start On Begin Play`，或在蓝图里调用 `Start Listening`。
6. **绑定事件 / Bind the event** — 监听 `On Wake Word Detected (Keyword)`，在回调里启动你的对话/交互逻辑。

```
[Event BeginPlay] ──▶ [WakeWord ▸ Start Listening]
[WakeWord ▸ On Wake Word Detected] ──(Keyword)──▶ [你的对话管线 / your dialog pipeline]
```

默认内置唤醒词 / Built-in default wake words:
- 中文 / Chinese (`kws-zh`): **你好双蛙**、**你好小蛙**、**小青蛙**
- 英文 / English (`kws-en`): **HELLO FROG**、**HEY FROG**、**OK COMPUTER**

---

## 中文还是英文？/ Chinese or English?

唤醒词的语言由**模型**决定（两个模型不能同时混用，一个组件只加载一个模型）：

| | 中文 / Chinese | 英文 / English |
|---|---|---|
| 模型目录 / Model dir | `Resources/kws-zh` (wenetspeech) | `Resources/kws-en` (gigaspeech) |
| 建模单元 / Units | 拼音 (声母 + 带调韵母) | sentencepiece BPE/unigram 子词 |
| 选择方式 / How to select | 项目设置 → Wake Word → `Language = Chinese` | `Language = English` |
| 分词 / Tokenization | 内置拼音字典，运行时自动 | 内置 sentencepiece 还原 (Viterbi)，运行时自动 |

- **组件层覆盖 / Per-component override**：组件 `Model → Preset` 可设 `Auto`（跟随项目设置，默认）/`Chinese`/`English`/`Custom`。
- 中、英文唤醒词都是**填原文即可**，引擎内自动转成模型需要的 token——中文用拼音、英文用 BPE，两者都与训练时一致，命中率一致地高。
- 想同时支持中英文？给一个 Actor 挂两个 `Wake Word Component`：一个 `Preset=Chinese`，一个 `Preset=English`，各绑各的事件即可。

---

## 架构 / Architecture

```
UE AudioCapture (默认麦克风, 48kHz float)
  → [音频线程] 降混单声道 + 线性重采样到 16kHz float
  → SPSC 无锁队列
  → [后台工作线程] sherpa-onnx KeywordSpotter 流式解码
  → 命中 → AsyncTask(GameThread) → 广播 OnWakeWordDetected
```

- 麦克风在 **GameThread** 打开/关闭（符合引擎约定）。
- 音频回调在 **音频设备线程**：降混 + 重采样 + 入队（单生产者）。
- **工作线程**（单消费者）：喂波形、解码、命中后 `Reset` 流，并通过 `AsyncTask` 回到 GameThread 广播。
- 模型目录 + 分词方案在 **GameThread** 解析快照后交给工作线程，工作线程不碰任何 UObject。
- 命中后有可配置的 **冷却时间**，避免同一次发声重复触发。

---

## 自定义唤醒词 / Custom keywords

### 推荐：面板/蓝图填原文即可，自动转换+保存 / Recommended: type the phrase — auto-converted & saved

**最省事**：先在面板选好 `Language`，再直接填**原文**（中文或英文），插件自动转 token 并持久化，**无需碰任何文件、无需 python**。
Easiest path: pick `Language`, then just enter the **phrase** (Chinese or English); tokens are generated and persisted automatically.

- **面板 / Panel**：项目设置 → Plugins → **Wake Word** → 在 `Wake Words` 里加一条，填 `Phrase`（中文如「芝麻开门」，英文如 `OPEN THE DOOR`），`Tokens` 自动显示。
- **蓝图 / Blueprint**：`Set Language` / `Add Wake Word` / `Remove Wake Word` / `Get Wake Words` / `Preview Tokens`（`UWakeWordLibrary`），改完调组件 `Refresh Wake Words` 生效。
- 详见 [`Docs/GitHub-Wiki.zh-CN.md`](Docs/GitHub-Wiki.zh-CN.md) / [`Docs/GitHub-Wiki.en.md`](Docs/GitHub-Wiki.en.md)。

> 英文小贴士：用 **2 个以上音节、发音清晰**的词更稳（如 `HELLO FROG`、`OK COMPUTER`）。极短的词（`GO`、`NO`）建议加 `:2.0` boosting。

### 进阶：手写原始 token / Advanced: hand-written raw tokens

每个模型用一行 token 定义一个唤醒词，格式相同：

```
<tokens>  [:boost] [#threshold]  @<显示名 / display name>
```

中文 (`Resources/kws-zh/keywords.txt`) — 拼音 token：

```
n ǐ h ǎo sh uāng w ā @你好双蛙
x iǎo q īng w ā :2.0 #0.20 @小青蛙
```

英文 (`Resources/kws-en/keywords.txt`) — BPE token（`▁` 是词首标记）：

```
▁HE LL O ▁F RO G @HELLO FROG
▁HE Y ▁F RO G :2.0 #0.20 @HEY FROG
▁O K ▁COMP U TER @OK COMPUTER
```

- `:2.0` — 该词的 **boosting 分数**（越大越容易命中，短词建议 2.0）。
- `#0.20` — 该词的 **独立阈值**（越大越保守，误唤醒少）。
- `@...` — 命中时回调里收到的字符串。

> 运行时也可不改文件：在组件的 `Custom Keywords (raw tokens)` 数组里填上面的行（非空时覆盖一切），
> 或在蓝图/C++ 调用 `Set Custom Keywords`。

### 用工具生成/校验 token / Generate / verify tokens with the helper

引擎内已自动分词，一般**无需**此工具。需要精确预生成或校验时：

```bash
cd Plugins/WakeWord/Tools

# 中文 / Chinese (pip install pypinyin)
python text2token.py --lang zh --tokens ../Resources/kws-zh/tokens.txt 你好双蛙 小青蛙

# 英文 / English (pip install sentencepiece) — 与运行时 C++ 分词器逐字一致
python text2token.py --lang en --bpe-model ../Resources/kws-en/bpe.model \
    --tokens ../Resources/kws-en/tokens.txt "HELLO FROG" "OK COMPUTER"
python text2token.py --lang en --bpe-model ../Resources/kws-en/bpe.model --self-test ../Resources/kws-en
```

---

## 调参经验 / Tuning

| 现象 / Symptom            | 调整 / Adjustment                                            |
|---------------------------|--------------------------------------------------------------|
| 误唤醒多 / false triggers | `Keywords Threshold` 0.25 → 0.35；或给该词加 `#0.30`          |
| 漏唤醒 / missed wakes      | 降低阈值；或给该词加 `:2.0` boosting；选更长/更清晰的词       |
| 短词容易误触              | 避免 1–2 音节词，用 3+ 音节（中文「你好双蛙」、英文 `OK COMPUTER`）|

- **中文**：用 4 音节词唤醒成功率最高（90%+）。
- **英文**：用内置英文模型 (`Language = English`)；选发音清晰、2+ 词的短语，极短词加 boosting。

---

## 接入对话管线 / Pairing with a dialog pipeline

为避免 KWS 与对话 ASR 抢麦克风，推荐"休眠—唤醒"循环：

```
OnWakeWordDetected ─▶ StopListening() ─▶ 启动实时对话会话(如 RealtimeVoice)
                                   ─▶ 对话结束(VAD 超时/挂断) ─▶ StartListening()
```

本插件与同作者的 **RealtimeVoice** / **MetaHumanAudioToFaceRuntime** 天然契合：
唤醒后把麦克风交给对话组件，对话结束再恢复监听。详见 [`Docs/Integration.md`](Docs/Integration.md)。

---

## 蓝图 API / Blueprint API

**事件 / Events**
- `On Wake Word Detected (Keyword: String)` — 命中唤醒词。
- `On Listening Started` — 麦克风已开、模型就绪。
- `On Error (Code: Int, Message: String)` — 出错（Code < 0 为客户端错误）。

**组件函数 / Component functions**
- `Start Listening` / `Stop Listening`
- `Is Listening` → bool
- `Is Ready` → bool（原生库已加载且模型文件齐全）
- `Get Model Dir` → String
- `Set Custom Keywords (Lines, bRestartIfListening)` / `Refresh Wake Words`

**库函数 / Library (`UWakeWordLibrary`)**
- `Get Language` / `Set Language (Language, bSave)` — 切换中文/英文。
- `Add Wake Word` / `Remove Wake Word` / `Clear Wake Words` / `Get Wake Words` / `Has Wake Word`
- `Preview Tokens` / `Save Wake Words`

**关键属性 / Key properties**
- `Model → Preset` — Auto（跟随项目语言，默认）/ Chinese / English / Custom。
- `Keywords Threshold` (0.05–0.5, 默认 0.25) / `Keywords Score` (默认 1.5)。
- `Auto Start On Begin Play` (默认 false) / `Retrigger Cooldown (s)` (默认 1.0)。
- `Use Project Wake Words` (默认 true) / `Custom Keywords (raw tokens)` / `Num Threads`（高级）。

完整说明见 [`Docs/BlueprintGuide.md`](Docs/BlueprintGuide.md)。

---

## 从源码构建 / Building from source

编辑器目标 / Editor target（**先关闭编辑器**，否则 MetaHuman DLL 占用会导致 LNK1104）：

```bat
"G:\games\UE_5.6\Engine\Build\BatchFiles\Build.bat" MyProject_bannEditor Win64 Development ^
  -Project="G:\ue56pj\MyProject_bann\MyProject_bann.uproject" -WaitMutex -NoHotReloadFromIDE
```

打包为可分发插件 / Package as a redistributable plugin（路径需 ASCII）：

```bat
"G:\games\UE_5.6\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin ^
  -Plugin="...\Plugins\WakeWord\WakeWord.uplugin" ^
  -Package="C:\PluginDist\WakeWord" -CreateSubFolder -TargetPlatforms=Win64
```

构建时 `WakeWord.Build.cs` 会自动把 `Source/ThirdParty/SherpaOnnx/bin/Win64` 下的所有 DLL
拷到模块 `Binaries/Win64`；模块启动时再从该目录显式加载（delay-load）。

> **UE 5.6 与 5.7**：源码同时支持两个引擎版本（引擎 API 稳定）。各自副本的差异只有 `WakeWord.uplugin`
> 里的 `EngineVersion` / `Installed` 字段。

---

## 目录结构 / Layout

```
WakeWord/
├─ WakeWord.uplugin
├─ README.md  LICENSE.txt  THIRD_PARTY_NOTICES.md
├─ Config/FilterPlugin.ini
├─ Docs/                         使用指南 / guides
├─ Resources/
│  ├─ Icon128.png
│  ├─ pinyin-zh.txt              中文 Hanzi→拼音 字典 (运行时分词)
│  ├─ kws-zh/                    中文模型 + tokens.txt + keywords.txt
│  └─ kws-en/                    英文模型 + tokens.txt + bpe.model + bpe-scores.txt + keywords.txt
├─ Tools/text2token.py          token 生成/校验器 (中文拼音 / 英文 BPE)
└─ Source/
   ├─ WakeWord/                  运行时模块 (组件 + 设置 + 库 + 分词器)
   └─ ThirdParty/SherpaOnnx/     头文件 / 导入库 / 运行时 DLL
```

---

## 测试与验证 / Verification

见 [`Docs/Testing.md`](Docs/Testing.md)。出厂验证包括：编辑器目标编译+链接通过；中、英文分词器输出与
sentencepiece / pypinyin 训练时分词**逐字一致**（`text2token.py --self-test`）；自定义 `keywords.txt` 解析无误且不误触。

---

## 许可 / License

插件本体版权归 odysseyzjh，见 [`LICENSE.txt`](LICENSE.txt)。内置的 sherpa-onnx / ONNX Runtime /
模型（wenetspeech、gigaspeech）为各自开源许可（Apache-2.0 / MIT），见 [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md)。
