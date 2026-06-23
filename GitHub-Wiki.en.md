# WakeWord Plugin · English Guide

Offline wake-word / keyword spotting for **Unreal Engine 5.6+**, powered by
[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (k2-fsa, Apache-2.0). It bundles two
3.3M models: **Chinese (wenetspeech)** and **English (gigaspeech)**.

> Platform: Win64 · Engine: UE 5.6 + · Version: 1.2.0 · Fully local, low latency, no
> per-call cost.

---

## Contents
1. [Install](#1-install)
2. [Quick Start](#2-quick-start)
3. [Chinese vs English](#3-chinese-vs-english)
4. [Managing Wake Words](#4-managing-wake-words)
5. [Tuning](#5-tuning)
6. [Dialog Pipeline](#6-dialog-pipeline)
7. [Blueprint / C++ API](#7-blueprint--c-api)
8. [Troubleshooting](#8-troubleshooting)
9. [FAQ](#9-faq)

---

## 1. Install

1. Copy the whole `WakeWord/` folder into your project's `Plugins/` directory.
2. Enable **Wake Word** in `Edit -> Plugins` or in your `.uproject`, then restart the editor.
3. Let Unreal compile the plugin. The build stages the bundled runtime DLLs into
   `Plugins/WakeWord/Binaries/Win64/`.

No model download is required. The plugin ships with the Chinese and English model files,
tokenizers, dictionaries, and Win64 runtime DLLs.

---

## 2. Quick Start

1. Create an Actor Blueprint, for example `BP_WakeWordManager`, and place it in the level.
2. Add **Wake Word Component**.
3. In the component Details panel, enable **Auto Start On Begin Play**.
4. Bind **On Wake Word Detected** and print the `Keyword` pin.
5. Press Play and say **"你好双蛙"**. The matched phrase should print.

Built-in default phrases:

| Language | Component preset | Default phrases |
|---|---|---|
| Chinese | `Model -> Preset = Chinese` or `Auto` with `Default Language = Chinese` | `你好双蛙`, `你好小蛙`, `小青蛙` |
| English | `Model -> Preset = English` or `Auto` with `Default Language = English` | `HELLO FROG`, `HEY FROG`, `OK COMPUTER` |

For English, set the component `Model -> Preset` to **English** and say **"HELLO FROG"**.

---

## 3. Chinese vs English

A wake-word model is single-language. WakeWord 1.2.0 therefore uses **two separate project lists**:

| | Chinese | English |
|---|---|---|
| Project list | `Chinese Wake Words` | `English Wake Words` |
| Model folder | `Resources/kws-zh` | `Resources/kws-en` |
| Tokenization | pinyin initials + finals | sentencepiece BPE/unigram |
| Component | `Preset=Chinese` | `Preset=English` |

- A `Preset=Chinese` component reads `Chinese Wake Words`.
- A `Preset=English` component reads `English Wake Words`.
- A `Preset=Auto` component follows **Default Language (Auto preset)**.
- To support both languages at once, add two Wake Word components: one `Preset=Chinese`, one
  `Preset=English`. Both can keep `Use Project Wake Words` enabled because they read different lists.

Do not put English phrases in the Chinese list or Chinese phrases in the English list. A wrong-language
phrase produces no valid tokens and is skipped with a warning instead of crashing the process.

---

## 4. Managing Wake Words

### Project Settings Panel

Open **Project Settings -> Plugins -> Wake Word**:

| Field | Meaning |
|---|---|
| `Chinese Wake Words` | Chinese phrases for the Chinese model. Type the original phrase; `Tokens` is generated automatically. |
| `English Wake Words` | English phrases for the English model. Type normal English words; `Tokens` is generated automatically. |
| `Boost (0 = global)` | Per-word boosting. Higher means easier to trigger. Try `2.0` for short phrases. |
| `Threshold (0 = global)` | Per-word threshold. Higher means more conservative. |
| `Default Threshold` | Global threshold when an entry leaves `Threshold` at `0`. |
| `Default Score` | Global boosting score when an entry leaves `Boost` at `0`. |
| `Default Language (Auto preset)` | Only affects components whose `Model -> Preset` is `Auto`. |

Edits persist to `Config/DefaultGame.ini`, using `+ChineseWakeWords=`, `+EnglishWakeWords=`, and
`DefaultLanguage=`.

If a component is already listening, call **Refresh Wake Words** or restart PIE to apply changes.

### Blueprint Runtime API

Every list operation takes a `Language` parameter:

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

Typical player-customization flow:

```text
Add Wake Word(Language=English, Phrase="OPEN THE DOOR", bSave=true)
    -> WakeWordComponent(Preset=English) -> Refresh Wake Words
```

### Raw Token Lines

Advanced users can bypass phrase tokenization with component **Custom Keywords (raw tokens)** or
`Set Custom Keywords`:

```text
n ǐ h ǎo sh uāng w ā @你好双蛙
▁HE LL O ▁F RO G @HELLO_FROG
```

Format:

```text
<tokens> [:boost] [#threshold] @display_id
```

WakeWord 1.2.0 validates tokens against the model `tokens.txt` before sherpa-onnx sees them. Invalid
lines are dropped and logged, which prevents bad custom keywords from killing the process. The plugin
also swaps spaces inside `@display_id` to `_` before sherpa and restores them in the detection callback,
but writing underscores yourself is still recommended for raw lines.

---

## 5. Tuning

| Symptom | Adjustment |
|---|---|
| False triggers | Raise `Default Threshold` from `0.25` to `0.35`, or set that entry's `Threshold` around `0.30`. |
| Missed wakes | Lower the threshold, raise `Boost`, and use a longer, clearer phrase. |
| Short phrase unstable | Prefer 3+ syllables/words; add `Boost = 2.0` for very short phrases. |

`Retrigger Cooldown (s)` controls how soon the same utterance may trigger again. The default is `1.0`.

---

## 6. Dialog Pipeline

WakeWord is the always-on switch. When it fires, stop KWS, hand the microphone to your dialog component,
then restart KWS when dialog ends:

```text
WakeWord.OnWakeWordDetected
    -> WakeWord.StopListening()
    -> RealtimeVoice.StartSession()

RealtimeVoice.OnSessionFinished
    -> RealtimeVoice.StopSession()
    -> WakeWord.StartListening()
```

Set RealtimeVoice `Auto Start On Begin Play` to false so it does not grab the microphone before the wake
word fires. See `Docs/Integration.md` for details.

---

## 7. Blueprint / C++ API

### Component Events

- `On Wake Word Detected(Keyword)` — a phrase was detected.
- `On Listening Started` — microphone and spotter are ready.
- `On Error(Code, Message)` — startup/runtime error.

### Component Functions

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

`SetDefaultLanguage(EWakeWordLanguage::English, true)` only changes components that use
`Model -> Preset = Auto`.

---

## 8. Troubleshooting

| Symptom | Check |
|---|---|
| `On Error` code `-10` | Native DLLs did not load. Confirm `Binaries/Win64` has `onnxrt_ww.dll` and `sherpa-onnx-c-api.dll`. |
| `On Error` code `-11` | Model or keyword source missing. Use `Get Model Dir` and confirm the model files and a matching wake-word list exist. |
| `On Error` code `-3` | Microphone cannot open. Check Windows mic permission and the default recording device. |
| No detection | Make sure the phrase is in the list matching the component preset; use `Preview Tokens`; lower the threshold. |
| `produced no tokens` warning | The phrase is in the wrong language list or contains unsupported characters. Move it to the correct list or change the phrase. |

---

## 9. FAQ

**Can Chinese and English run at the same time?**
Yes. Add two components, one `Preset=Chinese` and one `Preset=English`, and bind both events.

**Where are wake words saved?**
In `Config/DefaultGame.ini` when edited in the panel or saved with `bSave=true`.

**Does English really work?**
Yes. Put English phrases in `English Wake Words` and use a component with `Preset=English`. The bundled
sentencepiece-compatible tokenizer reproduces the model tokenization at runtime.

**Can I mix Chinese and English in one phrase?**
No. One model recognizes one token system. Mixed/wrong-language phrases are skipped. Use two components.

**Can I use a different model?**
Yes. Put the ONNX files and `tokens.txt` in a folder, set the component `Model -> Preset` to `Custom`,
and fill the folder/file names.

---

