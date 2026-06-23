# WakeWord 插件 · 中文教程

UE 5.6 + 离线唤醒词检测插件。基于
[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (k2-fsa, Apache-2.0) 的 Keyword Spotting，
内置 **中文 (wenetspeech 3.3M)** 与 **英文 (gigaspeech 3.3M)** 两套模型。

> 平台：Win64 · 引擎：UE 5.6 + · 版本：1.2.0 · 100% 本地、低延迟、无每次调用费用。

---

## 目录
1. [安装](#1-安装)
2. [快速开始](#2-快速开始)
3. [中文与英文](#3-中文与英文)
4. [管理唤醒词](#4-管理唤醒词)
5. [调参](#5-调参)
6. [对话管线接入](#6-对话管线接入)
7. [蓝图 / C++ API](#7-蓝图--c-api)
8. [排错](#8-排错)
9. [FAQ](#9-faq)

---

## 1. 安装

1. 把整个 `WakeWord/` 文件夹复制到工程的 `Plugins/` 目录。
2. 在 `Edit -> Plugins` 或 `.uproject` 中启用 **Wake Word**，然后重启编辑器。
3. 让 Unreal 编译插件。构建会把内置运行时 DLL 自动放到 `Plugins/WakeWord/Binaries/Win64/`。

插件自带中文/英文模型、tokens、分词资源和 Win64 DLL，无需额外下载模型。

---

## 2. 快速开始

1. 新建一个 Actor 蓝图，例如 `BP_WakeWordManager`，拖进关卡。
2. 添加 **Wake Word Component**。
3. 在组件 Details 面板里勾选 **Auto Start On Begin Play**。
4. 绑定 **On Wake Word Detected**，把 `Keyword` 接到 `Print String`。
5. Play 后对麦克风说 **"你好双蛙"**，屏幕应打印命中的短语。

内置默认词：

| 语言 | 组件 Preset | 默认唤醒词 |
|---|---|---|
| 中文 | `Model -> Preset = Chinese`，或 `Auto` + `Default Language = Chinese` | `你好双蛙`、`你好小蛙`、`小青蛙` |
| 英文 | `Model -> Preset = English`，或 `Auto` + `Default Language = English` | `HELLO FROG`、`HEY FROG`、`OK COMPUTER` |

要测试英文，把组件 `Model -> Preset` 设为 **English**，然后说 **"HELLO FROG"**。

---

## 3. 中文与英文

一个唤醒词模型只能识别一种 token 系统。WakeWord 1.2.0 因此把项目设置拆成**两个独立列表**：

| | 中文 | 英文 |
|---|---|---|
| 项目列表 | `Chinese Wake Words` | `English Wake Words` |
| 模型目录 | `Resources/kws-zh` | `Resources/kws-en` |
| 分词方式 | 拼音声母 + 韵母 | sentencepiece BPE/unigram |
| 组件选择 | `Preset=Chinese` | `Preset=English` |

- `Preset=Chinese` 的组件读取 `Chinese Wake Words`。
- `Preset=English` 的组件读取 `English Wake Words`。
- `Preset=Auto` 的组件跟随 **Default Language (Auto preset)**。
- 中英文同时常驻时，给一个 Actor 挂两个 Wake Word Component：一个 `Preset=Chinese`，一个 `Preset=English`。两个组件都可以勾 `Use Project Wake Words`，因为它们会各自读对应列表。

不要把英文短语放进中文列表，也不要把中文短语放进英文列表。放错语言会生成不了有效 token，插件会跳过并打印警告，而不是让 sherpa-onnx 崩溃。

---

## 4. 管理唤醒词

### 项目设置面板

打开 **Project Settings -> Plugins -> Wake Word**：

| 字段 | 说明 |
|---|---|
| `Chinese Wake Words` | 中文模型的短语列表。填原文，`Tokens` 自动生成。 |
| `English Wake Words` | 英文模型的短语列表。填正常英文，`Tokens` 自动生成。 |
| `Boost (0 = global)` | 单词 boosting。越大越容易命中，短词可试 `2.0`。 |
| `Threshold (0 = global)` | 单词独立阈值。越大越保守。 |
| `Default Threshold` | 条目 `Threshold=0` 时使用的全局阈值。 |
| `Default Score` | 条目 `Boost=0` 时使用的全局 boosting。 |
| `Default Language (Auto preset)` | 只影响 `Model -> Preset = Auto` 的组件。 |

面板修改会持久化到 `Config/DefaultGame.ini`，对应键包括 `+ChineseWakeWords=`、
`+EnglishWakeWords=` 和 `DefaultLanguage=`。

如果组件正在监听，修改后调用 **Refresh Wake Words** 或重开 PIE 即可生效。

### 蓝图运行时 API

所有列表操作都带 `Language` 参数：

```text
Add Wake Word(Language, Phrase, Boost, Threshold, bSave)
Remove Wake Word(Language, Phrase, bSave)
Clear Wake Words(Language, bSave)
Get Wake Words(Language)
Has Wake Word(Language, Phrase)
Preview Tokens(Language, Phrase)
Save Wake Words()
Get Default Language()
Set Default Language(Language, bSave)
```

典型的玩家自定义流程：

```text
Add Wake Word(Language=English, Phrase="OPEN THE DOOR", bSave=true)
    -> WakeWordComponent(Preset=English) -> Refresh Wake Words
```

### 原始 token 行

高级用户可以通过组件 **Custom Keywords (raw tokens)** 或 `Set Custom Keywords` 绕过自动分词：

```text
n ǐ h ǎo sh uāng w ā @你好双蛙
▁HE LL O ▁F RO G @HELLO_FROG
```

格式：

```text
<tokens> [:boost] [#threshold] @display_id
```

WakeWord 1.2.0 会在交给 sherpa-onnx 前用模型的 `tokens.txt` 校验每一行。含未知 token 的行会被丢弃并写日志，避免坏关键词直接杀进程。插件也会把 `@display_id` 内部的空格换成 `_`，命中回调时再换回空格；手写 raw 行时仍建议自己写下划线。

---

## 5. 调参

| 现象 | 调整 |
|---|---|
| 误唤醒多 | `Default Threshold` 从 `0.25` 调到 `0.35`，或给该词设 `Threshold=0.30` 左右。 |
| 漏唤醒 | 降低阈值、提高 `Boost`，并使用更长更清晰的短语。 |
| 短词不稳定 | 优先 3 个音节/词以上；非常短的词加 `Boost=2.0`。 |

`Retrigger Cooldown (s)` 控制一次命中后多久允许再次触发，默认 `1.0` 秒。

---

## 6. 对话管线接入

WakeWord 适合作为常驻开关。命中后停止 KWS，把麦克风交给对话组件；对话结束再恢复 KWS：

```text
WakeWord.OnWakeWordDetected
    -> WakeWord.StopListening()
    -> RealtimeVoice.StartSession()

RealtimeVoice.OnSessionFinished
    -> RealtimeVoice.StopSession()
    -> WakeWord.StartListening()
```

把 RealtimeVoice 的 `Auto Start On Begin Play` 设为 false，避免它一进场就抢麦克风。详见
`Docs/Integration.md`。

---

## 7. 蓝图 / C++ API

### 组件事件

- `On Wake Word Detected(Keyword)`：检测到唤醒词。
- `On Listening Started`：麦克风打开，spotter 已就绪。
- `On Error(Code, Message)`：启动或运行错误。

### 组件函数

- `Start Listening` / `Stop Listening` / `Is Listening`
- `Is Ready`
- `Refresh Wake Words`
- `Set Custom Keywords`
- `Get Model Dir`

### C++

```cpp
#include "WakeWordComponent.h"
#include "WakeWordLibrary.h"
#include "WakeWordSettings.h"

UWakeWordLibrary::AddWakeWord(EWakeWordLanguage::Chinese, TEXT("芝麻开门"), 0.f, 0.f, true);
UWakeWordLibrary::AddWakeWord(EWakeWordLanguage::English, TEXT("OPEN THE DOOR"), 0.f, 0.f, true);

WakeWord->Model.Preset = EWakeWordModelPreset::English;
WakeWord->RefreshWakeWords();
WakeWord->StartListening();
```

`SetDefaultLanguage(EWakeWordLanguage::English, true)` 只影响 `Model -> Preset = Auto` 的组件。

---

## 8. 排错

| 现象 | 排查 |
|---|---|
| `On Error` code `-10` | 原生 DLL 没加载。确认 `Binaries/Win64` 下有 `onnxrt_ww.dll` 与 `sherpa-onnx-c-api.dll`。 |
| `On Error` code `-11` | 模型或关键词源缺失。用 `Get Model Dir` 看路径，确认模型文件和当前 Preset 对应的唤醒词列表存在。 |
| `On Error` code `-3` | 麦克风打不开。检查 Windows 麦克风权限和默认录音设备。 |
| 一直无命中 | 确认短语在组件 Preset 对应的列表里；用 `Preview Tokens` 检查；适当降低阈值。 |
| `produced no tokens` 警告 | 短语放错语言列表，或包含不支持字符。移到正确列表或换个词。 |

---

## 9. FAQ

**Q：能中英文同时用吗？**
A：可以。挂两个组件，一个 `Preset=Chinese`，一个 `Preset=English`，各绑各的事件。

**Q：唤醒词存在哪里？**
A：面板修改或 `bSave=true` 时写入项目的 `Config/DefaultGame.ini`。

**Q：英文识别可用吗？**
A：可用。英文短语填 `English Wake Words`，组件设 `Preset=English`。内置 sentencepiece 兼容分词器会在运行时生成模型需要的 BPE token。

**Q：一条短语里能中英混合吗？**
A：不建议，也不再做英文逐字母兜底。一个模型只识别一种 token 系统；要中英文都可用，请用两个组件。

**Q：能换自己的模型吗？**
A：可以。把 ONNX 文件和 `tokens.txt` 放到目录里，组件 `Model -> Preset` 设为 `Custom`，再填写目录和文件名。

---

