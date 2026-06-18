# WakeWord 插件 · 中文教程

UE 5.6 / 5.7 离线唤醒词检测插件。基于 [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx)(k2-fsa, Apache-2.0) 的 Keyword Spotting，内置 **中文 (wenetspeech 3.3M)** 与 **英文 (gigaspeech 3.3M)** 两个模型，一个开关切换。**100% 本地、低延迟、无每次调用费用**。

> 平台：Win64 · 引擎：UE 5.6 / 5.7 · 唤醒词：先选语言(中文/英文)，再在面板或蓝图里填原文即可，自动转换、自动保存。

---

## 目录
1. [安装](#1-安装)
2. [5 分钟上手](#2-5-分钟上手)
3. [管理唤醒词（重点）](#3-管理唤醒词重点)
4. [调参](#4-调参)
5. [接入对话管线（RealtimeVoice）](#5-接入对话管线realtimevoice)
6. [蓝图 / C++ API](#6-蓝图--c-api)
7. [排错](#7-排错)
8. [常见问题 FAQ](#8-常见问题-faq)

---

## 1. 安装

1. 把 `WakeWord/` 整个文件夹复制到工程的 `Plugins/` 目录。
2. 打开工程（或 `Edit → Plugins` 勾选 **Wake Word** 并重启），让它编译。
3. 确认 `Plugins/WakeWord/Binaries/Win64/` 下生成了 `onnxrt_ww.dll` 与 `sherpa-onnx-c-api.dll`（构建会自动拷贝）。

> 插件已自带模型与运行时 DLL，无需任何额外下载或安装。

---

## 2. 5 分钟上手

1. 新建一个 **Actor 蓝图**（如 `BP_WakeWordManager`），拖进关卡。
2. **Components → + Add → 搜 "Wake Word" → 添加 Wake Word Component**。
3. 选中组件，Details 面板里勾上 **`Auto Start On Begin Play`**。
4. 在 Components 面板选中该组件 → Details 最下方 **Events** → 点 `On Wake Word Detected` 旁的绿色 **＋**，生成事件节点。
5. 把事件的 `Keyword` 引脚接到 `Print String`。
6. Play，对麦克风说 **"你好双蛙"** → 屏幕打印命中的词。

就这么简单。默认唤醒词是 **你好双蛙 / 小青蛙**（中文），英文模型默认 **HELLO FROG / OK COMPUTER**，可随时改（见下一节）。

> 想要英文？项目设置 → Plugins → Wake Word → **Language = English**，对麦克风说 **"HELLO FROG"** 即可。详见下一节。

---

## 3. 管理唤醒词（重点）

### 先选语言：中文 or 英文

唤醒词的语言由**模型**决定，两个模型不能混用（一个组件只加载一个模型）：

| | 中文 | 英文 |
|---|---|---|
| 模型 | wenetspeech (`Resources/kws-zh`) | gigaspeech (`Resources/kws-en`) |
| 建模单元 | 拼音(声母+韵母) | sentencepiece BPE 子词 |
| 怎么选 | 项目设置 → Wake Word → `Language = Chinese` | `Language = English` |

- 组件 `Model → Preset` 默认 **Auto**，自动跟随上面的 `Language`。也可在组件上强制 `Chinese` / `English` / `Custom`。
- 切换 `Language` 时，若你**没改过**唤醒词列表，会自动换成该语言的默认词（中文「你好双蛙…」/ 英文「HELLO FROG…」）；**改过的列表不会被动**。
- **想中英同时支持**：给一个 Actor 挂两个 `Wake Word Component`，一个 `Preset=Chinese`、一个 `Preset=English`，各绑各的事件（第二种语言的组件别勾 `Use Project Wake Words`，用它自己的 `Custom Keywords` 或内置 keywords.txt）。

选好语言后，你只需要填**原文**（中文或英文），插件会自动转成该模型需要的 token 并保存——**不用碰任何文件、不用跑任何脚本**。两个入口：

### 方式 A：项目设置面板（推荐，适合策划/美术）

**项目设置 → Plugins → Wake Word**：

| 字段 | 说明 |
|---|---|
| **Language** | 唤醒词语言：`Chinese`(kws-zh) 或 `English`(kws-en)。决定加载哪个模型与如何分词。 |
| **Wake Words** | 唤醒词数组。点 ＋ 加一条，填 `Phrase`（中文"芝麻开门"，英文 `OPEN THE DOOR`）。`Tokens` 会按当前 Language 自动显示生成的 token，方便核对。 |
| **Boost (0=global)** | 该词的命中加权，0=用全局值；短词可设 2.0。 |
| **Threshold (0=global)** | 该词的独立阈值，0=用全局值。 |
| **Default Threshold** | 全局灵敏度（误唤醒多→调高到 0.35）。 |
| **Default Score** | 全局 boosting。 |

填好后会**自动持久化**到 `Config/DefaultGame.ini`，项目级生效。组件默认 `Use Project Wake Words = true`，直接读这里。

> 改完后如果组件正在监听，调用组件的 **`Refresh Wake Words`**（或重开 PIE）即可生效。

### 方式 B：蓝图运行时增删查（适合做"让玩家自定义唤醒词"的 UI）

`UWakeWordLibrary` 提供全局函数（右键搜节点名）：

```
Set Language (Language=English, bSave=true)   // 切中文/英文(切换模型与分词)
Get Language              -> 当前语言(Chinese / English)
Add Wake Word (Phrase="芝麻开门", Boost=0, Threshold=0, bSave=true)
Remove Wake Word (Phrase="芝麻开门")
Clear Wake Words
Get Wake Words            -> 当前所有唤醒词原文(数组)
Has Wake Word (Phrase)    -> 是否已存在
Preview Tokens (Phrase)   -> 预览会生成的 token(按当前语言, 不改列表)
Save Wake Words           -> 手动持久化
```

典型流程（玩家在 UI 里输入一个词并保存）：
```
[输入框文本] ─▶ Add Wake Word (bSave=true) ─▶ WakeWordComponent ▸ Refresh Wake Words
```

`bSave=true` 会写盘持久化，下次启动仍在。改完调用组件的 `Refresh Wake Words` 立即生效。

### 关于英文唤醒词

- **首选英文模型**：`Language = English`（或组件 `Preset = English`）。它用 gigaspeech 英文模型，识别率与中文模型相当。直接填英文原文（如 `OPEN THE DOOR`、`HELLO FROG`），插件内置 sentencepiece 分词器会**逐字一致地**还原模型需要的 BPE token，无需 python、无需改文件。
- **选词建议**：用发音清晰、**2 个词以上**的短语更稳；极短词（`GO`、`NO`）建议加 `Boost = 2.0`。大小写无所谓（内部统一大写）。
- **中英不能混在一条里**：中文模型只认拼音、英文模型只认 BPE，一个模型不能同时识别中英文。要两种语言同时可用，就挂两个组件（见上"先选语言")。
- **精确校验**：想确认某个英文词生成的 token，用 `Preview Tokens`，或离线运行 `Tools/text2token.py --lang en`（见 README）。

### 关于中英混合（仅中文模型）

中文模型下，中英混合短语（如"你好Jarvis"）：中文部分自动转拼音，英文字母按字母 token 处理（尽力而为）。纯英文请改用英文模型。

### 优先级

`Custom Keywords (raw tokens)`（组件高级项，原始 token）＞ 项目设置唤醒词 ＞ 内置 `keywords.txt`。

---

## 4. 调参

| 现象 | 调整 |
|---|---|
| 误唤醒多 | `Default Threshold` 0.25 → 0.35；或给该词设 `Threshold = 0.30` |
| 漏唤醒 | 降低阈值；或给该词设 `Boost = 2.0`；尽量用 4 音节以上的词 |
| 短词易误触 | 避免 2 音节（"小蛙"），用 4 音节（"你好双蛙"） |

`Retrigger Cooldown (s)`：命中后的冷却时间，默认 1 秒，防止一次发声重复触发。

---

## 5. 接入对话管线（RealtimeVoice）

唤醒词只是"开关"。命中后让出麦克风给对话组件，对话结束再恢复监听：

**前置设置**
- RealtimeVoice 组件的 `Auto Start On Begin Play` 设为 **false**（否则一进 Play 就抢麦）。
- WakeWord 组件的 `Auto Start On Begin Play` 设为 **true**。

**接线**
```
WakeWord.OnWakeWordDetected   ─▶ WakeWord.StopListening()  ─▶ RealtimeVoice.StartSession()
RealtimeVoice.OnSessionFinished ─▶ RealtimeVoice.StopSession() ─▶ WakeWord.StartListening()
```
两者都用默认麦克风、停止时彻底释放设备，顺序交接干净；无原生库冲突。

---

## 6. 蓝图 / C++ API

### 组件事件
- `On Wake Word Detected (Keyword: String)` — 命中唤醒词。
- `On Listening Started` — 麦克风已开、模型就绪。
- `On Error (Code: Int, Message: String)` — 出错（Code<0=客户端）。

### 组件函数
- `Start Listening` / `Stop Listening` / `Is Listening`
- `Is Ready` — 原生库已加载且模型/唤醒词齐全。
- `Refresh Wake Words` — 改完唤醒词后调用以立即生效。
- `Set Custom Keywords (Lines, bRestart)` — 高级：直接给原始 token 行。
- `Get Model Dir` — 排查路径用。

### 唤醒词库（UWakeWordLibrary，全局）
`Get / Set Language`、`Add / Remove / Clear / Get / Has Wake Word`、`Preview Tokens`、`Save Wake Words`。

### C++
```cpp
#include "WakeWordComponent.h"
#include "WakeWordLibrary.h"
#include "WakeWordSettings.h"   // EWakeWordLanguage

// 中文
UWakeWordLibrary::AddWakeWord(TEXT("芝麻开门"), 0.f, 0.f, /*bSave*/true);

// 英文：先切语言再加词
UWakeWordLibrary::SetLanguage(EWakeWordLanguage::English, /*bSave*/true);
UWakeWordLibrary::AddWakeWord(TEXT("OPEN THE DOOR"), 0.f, 0.f, true);

WakeWord->OnWakeWordDetected.AddDynamic(this, &AMyActor::OnWake);
WakeWord->RefreshWakeWords();   // 组件 Model.Preset=Auto 会跟随上面的语言
WakeWord->StartListening();
```

---

## 7. 排错

| 现象 | 排查 |
|---|---|
| `On Error` code `-10` | 原生 DLL 没加载：确认 `Binaries/Win64` 下有 `onnxrt_ww.dll`/`sherpa-onnx-c-api.dll`。 |
| `On Error` code `-11` | 模型/唤醒词缺失：用 `Get Model Dir` 看路径，确认 `Resources/kws-zh`（英文则 `kws-en`）文件齐全，且至少有一个唤醒词。 |
| `On Error` code `-3` | 麦克风打不开：检查默认录音设备与 Windows 麦克风权限。 |
| 一直无命中 | 看 Output Log 的 `LogWakeWord`：应有 `Spotter ready ...`；用 `Preview Tokens` 确认你的词生成的 token 正常；适当降阈值。 |
| 某词 `produced no tokens` 警告 | 该短语含无法识别的字符（生僻字/纯符号）；换个词或用 raw tokens。 |

---

## 8. 常见问题 FAQ

**Q：唤醒词存在哪？会不会丢？**
A：存在项目的 `Config/DefaultGame.ini`（面板填写或 `bSave=true` 时）。随工程走，不会丢。

**Q：要联网吗？有调用费用吗？**
A：完全离线，零费用，零隐私外泄。

**Q：能识别多少个唤醒词？**
A：数量不限，每个模型才 3.3M，开销可忽略。建议同时启用的别太多以控制误唤醒。

**Q：英文到底行不行？**
A：行。把 `Language` 切成 **English** 即用内置 gigaspeech 英文模型，直接填英文原文，识别率与中文相当。插件内置 sentencepiece 分词器与训练时分词逐字一致，无需 python。

**Q：能中英文同时用吗？**
A：一个模型只能识别一种语言。要同时支持，就在一个 Actor 上挂两个 `Wake Word Component`，一个 `Preset=Chinese`、一个 `Preset=English`，各绑各的事件。

**Q：换更高精度的模型行吗？**
A：把对应 onnx 放进 `Resources/kws-zh` 或 `kws-en`，把组件 `Model → Preset` 设为 `Custom` 并填目录/文件名即可（默认用 int8 小模型）。

---

许可：插件本体版权归 odysseyzjh；内置 sherpa-onnx / ONNX Runtime / 模型为各自开源许可（Apache-2.0 / MIT），见 `THIRD_PARTY_NOTICES.md`。
