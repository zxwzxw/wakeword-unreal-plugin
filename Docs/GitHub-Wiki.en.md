# WakeWord Plugin · English Guide

Offline wake-word / keyword spotting for **Unreal Engine 5.6 / 5.7**, powered by
[sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) (k2-fsa, Apache-2.0). Bundles **two 3.3M
models — Chinese (wenetspeech) and English (gigaspeech)** — switchable with one setting.
**100% on-device, low latency, no per-call cost.**

> Platform: Win64 · Engine: UE 5.6 / 5.7 · Wake words: pick a language (Chinese/English), then
> just type the phrase in a panel or Blueprint — it is auto-converted to model tokens and auto-saved.

---

## Contents
1. [Install](#1-install)
2. [Quick start (5 min)](#2-quick-start-5-min)
3. [Managing wake words (key feature)](#3-managing-wake-words-key-feature)
4. [Tuning](#4-tuning)
5. [Pairing with a dialog pipeline (RealtimeVoice)](#5-pairing-with-a-dialog-pipeline-realtimevoice)
6. [Blueprint / C++ API](#6-blueprint--c-api)
7. [Troubleshooting](#7-troubleshooting)
8. [FAQ](#8-faq)

---

## 1. Install

1. Copy the whole `WakeWord/` folder into your project's `Plugins/` directory.
2. Open the project (or enable **Wake Word** under `Edit → Plugins` and restart) to compile it.
3. Confirm `Plugins/WakeWord/Binaries/Win64/` contains `onnxrt_ww.dll` and
   `sherpa-onnx-c-api.dll` (the build stages them automatically).

> The model and runtime DLLs are bundled — no extra downloads or installs needed.

---

## 2. Quick start (5 min)

1. Create an **Actor Blueprint** (e.g. `BP_WakeWordManager`) and drop it in the level.
2. **Components → + Add → search "Wake Word" → add Wake Word Component**.
3. Select the component, tick **`Auto Start On Begin Play`** in Details.
4. With the component selected, scroll Details to **Events** → click the green **＋** next to
   `On Wake Word Detected` to spawn the event node.
5. Wire the event's `Keyword` pin into a `Print String`.
6. Press Play and say a wake word into your mic → the matched phrase prints.

Default wake words are **你好双蛙 / 小青蛙** (Chinese); the English model defaults to
**HELLO FROG / OK COMPUTER**. Change them any time (next section).

> Want English? Project Settings → Plugins → Wake Word → **Language = English**, then say
> **"HELLO FROG"**. See the next section.

---

## 3. Managing wake words (key feature)

### First pick a language: Chinese or English

The wake-word language is decided by the **model**; the two cannot be mixed (a component loads
exactly one model):

| | Chinese | English |
|---|---|---|
| Model | wenetspeech (`Resources/kws-zh`) | gigaspeech (`Resources/kws-en`) |
| Units | pinyin (initial + toned final) | sentencepiece BPE sub-words |
| How to select | Project Settings → Wake Word → `Language = Chinese` | `Language = English` |

- The component's `Model → Preset` defaults to **Auto**, which follows the `Language` above.
  You can also force `Chinese` / `English` / `Custom` per component.
- Switching `Language` auto-loads that language's default words **if you haven't customized the
  list** (Chinese "你好双蛙…" / English "HELLO FROG…"); a customized list is never touched.
- **Want both at once?** Put two `Wake Word Component`s on one Actor — one `Preset=Chinese`,
  one `Preset=English` — and bind each event (leave the second one's `Use Project Wake Words`
  off; give it its own `Custom Keywords` or the bundled keywords.txt).

After picking a language, you only enter the **phrase** (Chinese or English) — the plugin
auto-converts it to that model's tokens and persists it. **No file editing, no scripts.**
Two entry points:

### Option A — Project Settings panel (recommended)

**Project Settings → Plugins → Wake Word**:

| Field | Meaning |
|---|---|
| **Language** | Wake-word language: `Chinese` (kws-zh) or `English` (kws-en). Selects the model + tokenizer. |
| **Wake Words** | Array of entries. Click ＋, fill `Phrase` (Chinese "芝麻开门", English `OPEN THE DOOR`). `Tokens` auto-displays the generated tokens (per the current Language) so you can verify. |
| **Boost (0=global)** | Per-word boosting; 0 = use global. Use ~2.0 for short words. |
| **Threshold (0=global)** | Per-word threshold; 0 = use global. |
| **Default Threshold** | Global sensitivity (raise to 0.35 if you get false triggers). |
| **Default Score** | Global boosting. |

Edits auto-persist to `Config/DefaultGame.ini` (project-wide). The component reads this by
default (`Use Project Wake Words = true`).

> After editing, call the component's **`Refresh Wake Words`** (or restart PIE) to apply live.

### Option B — Blueprint runtime add/remove (e.g. let players define their own)

`UWakeWordLibrary` exposes global functions:

```
Set Language (Language=English, bSave=true)   // switch Chinese/English (model + tokenizer)
Get Language              -> current language (Chinese / English)
Add Wake Word (Phrase="芝麻开门", Boost=0, Threshold=0, bSave=true)
Remove Wake Word (Phrase="芝麻开门")
Clear Wake Words
Get Wake Words            -> all current phrases (array)
Has Wake Word (Phrase)    -> already exists?
Preview Tokens (Phrase)   -> preview generated tokens (current language, no change)
Save Wake Words           -> persist manually
```

Typical flow (player types a phrase and saves it):
```
[TextBox text] ─▶ Add Wake Word (bSave=true) ─▶ WakeWordComponent ▸ Refresh Wake Words
```

`bSave=true` writes to disk so it survives restarts. `Refresh Wake Words` applies it live.

### About English wake words

- **Use the English model**: `Language = English` (or component `Preset = English`). It uses the
  gigaspeech English model with accuracy on par with the Chinese one. Just type the English phrase
  (e.g. `OPEN THE DOOR`, `HELLO FROG`); the built-in sentencepiece tokenizer reproduces the BPE
  tokens the model expects **exactly** — no Python, no file editing. Case doesn't matter
  (it is upper-cased internally).
- **Choosing words**: clear, **2+ word** phrases are most reliable; add `Boost = 2.0` to very short
  words (`GO`, `NO`).
- **No mixing in one phrase**: the Chinese model only knows pinyin and the English model only knows
  BPE; one model can't do both languages. For both, use two components (see "First pick a language").
- **Exact verification**: use `Preview Tokens`, or run `Tools/text2token.py --lang en` offline (see README).

### About Chinese + Latin mixing (Chinese model only)

Under the Chinese model, a mixed phrase (e.g. "你好Jarvis") tokenizes the Chinese to pinyin and
maps Latin letters to letter tokens (best-effort). For pure English, switch to the English model.

### Priority

`Custom Keywords (raw tokens)` (component, advanced) > Project-settings wake words > bundled `keywords.txt`.

---

## 4. Tuning

| Symptom | Fix |
|---|---|
| False triggers | `Default Threshold` 0.25 → 0.35; or set the entry's `Threshold = 0.30` |
| Missed wakes | Lower the threshold; or set `Boost = 2.0`; prefer 4+ syllable phrases |
| Short word mis-fires | Avoid 2-syllable words; use 4 syllables (e.g. "你好双蛙") |

`Retrigger Cooldown (s)`: post-hit cooldown (default 1s) to prevent a single utterance
re-triggering.

---

## 5. Pairing with a dialog pipeline (RealtimeVoice)

A wake word is just a switch. On a hit, free the mic for the dialog component, then resume:

**Setup**
- Set RealtimeVoice's `Auto Start On Begin Play` to **false** (else it grabs the mic at BeginPlay).
- Set WakeWord's `Auto Start On Begin Play` to **true**.

**Wiring**
```
WakeWord.OnWakeWordDetected   ─▶ WakeWord.StopListening()   ─▶ RealtimeVoice.StartSession()
RealtimeVoice.OnSessionFinished ─▶ RealtimeVoice.StopSession() ─▶ WakeWord.StartListening()
```
Both use the default mic and fully release it on stop, so the hand-off is clean; no native-lib clash.

---

## 6. Blueprint / C++ API

### Component events
- `On Wake Word Detected (Keyword: String)` — a wake word fired.
- `On Listening Started` — mic open, spotter ready.
- `On Error (Code: Int, Message: String)` — error (Code < 0 = client side).

### Component functions
- `Start Listening` / `Stop Listening` / `Is Listening`
- `Is Ready` — native libs loaded and model/keywords present.
- `Refresh Wake Words` — apply wake-word changes live.
- `Set Custom Keywords (Lines, bRestart)` — advanced raw token lines.
- `Get Model Dir` — for path debugging.

### Wake-word library (UWakeWordLibrary, global)
`Get / Set Language`, `Add / Remove / Clear / Get / Has Wake Word`, `Preview Tokens`, `Save Wake Words`.

### C++
```cpp
#include "WakeWordComponent.h"
#include "WakeWordLibrary.h"
#include "WakeWordSettings.h"   // EWakeWordLanguage

// Chinese
UWakeWordLibrary::AddWakeWord(TEXT("芝麻开门"), 0.f, 0.f, /*bSave*/true);

// English: switch the language first, then add words
UWakeWordLibrary::SetLanguage(EWakeWordLanguage::English, /*bSave*/true);
UWakeWordLibrary::AddWakeWord(TEXT("OPEN THE DOOR"), 0.f, 0.f, true);

WakeWord->OnWakeWordDetected.AddDynamic(this, &AMyActor::OnWake);
WakeWord->RefreshWakeWords();   // component Model.Preset=Auto follows the language above
WakeWord->StartListening();
```

---

## 7. Troubleshooting

| Symptom | Check |
|---|---|
| `On Error` code `-10` | Native DLLs not loaded: confirm `Binaries/Win64` has `onnxrt_ww.dll` / `sherpa-onnx-c-api.dll`. |
| `On Error` code `-11` | Model/keywords missing: use `Get Model Dir`, confirm `Resources/kws-zh` (or `kws-en` for English) files exist and at least one wake word is defined. |
| `On Error` code `-3` | Mic won't open: check the default recording device and Windows mic permission. |
| Never detects | Check `LogWakeWord` for `Spotter ready ...`; use `Preview Tokens` to verify your phrase tokenizes; lower the threshold. |
| `produced no tokens` warning | The phrase has unsupported characters (rare glyphs / pure symbols); pick another phrase or use raw tokens. |

---

## 8. FAQ

**Q: Where are wake words stored? Do they persist?**
A: In the project's `Config/DefaultGame.ini` (when edited in the panel or saved with `bSave=true`). They travel with the project and survive restarts.

**Q: Does it need the internet? Any cost?**
A: Fully offline, zero cost, no data leaves the machine.

**Q: How many wake words can I have?**
A: Unlimited; each model is only 3.3M and the cost is negligible. Keep the active set modest to limit false triggers.

**Q: Does English actually work?**
A: Yes. Set `Language = English` to use the bundled gigaspeech English model and just type English phrases — accuracy is on par with Chinese. The built-in sentencepiece tokenizer matches the model's training tokenization exactly, no Python required.

**Q: Can I use Chinese and English at the same time?**
A: One model handles one language. For both, put two `Wake Word Component`s on one Actor — one `Preset=Chinese`, one `Preset=English` — and bind each event.

**Q: Can I swap in a higher-accuracy model?**
A: Drop the onnx files into `Resources/kws-zh` or `kws-en`, set the component's `Model → Preset` to `Custom`, and fill the folder/file names (defaults use the small int8 model).

---

License: the plugin is © odysseyzjh; bundled sherpa-onnx / ONNX Runtime / model are under their
own open-source licenses (Apache-2.0 / MIT) — see `THIRD_PARTY_NOTICES.md`.
