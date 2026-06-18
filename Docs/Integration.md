# Integration Guide / 对话管线接入

唤醒词检测（KWS）与对话 ASR 都要读麦克风。为避免抢占同一音频流，推荐"休眠—唤醒"循环：
平时只跑 KWS（开销可忽略），命中后把麦克风让给对话组件，对话结束再恢复监听。

## 推荐时序 / Recommended sequence

```
        ┌─────────────────────────── 监听中 / Listening ───────────────────────────┐
        │  WakeWord ▸ Start Listening                                               │
        ▼                                                                           │
  On Wake Word Detected (Keyword)                                                   │
        │                                                                           │
        ├─▶ WakeWord ▸ Stop Listening      (释放麦克风 / free the mic)              │
        │                                                                           │
        ├─▶ 启动对话会话 / start dialog (RealtimeVoice ▸ Start Session, 等等)       │
        │                                                                           │
        └─▶ 对话结束 (VAD 超时 / 用户挂断 / Session Finished)                       │
                 │                                                                  │
                 └─▶ WakeWord ▸ Start Listening  ───────────────────────────────────┘
```

要点 / Notes：
- **先 `Stop Listening` 再启动对话**，确保麦克风设备被 WakeWord 释放后再被对话组件打开。
- 对话组件结束的回调里再 `Start Listening`。若你的对话组件在 EndPlay 才关麦，记得在它**完全停止后**再恢复。

## 与 RealtimeVoice 配合 / With RealtimeVoice

> 以下基于 `Plugins/RealtimeVoice` 的实际 API 校对。两者均为 Win64 运行时模块，
> 都依赖引擎 `AudioCapture` 插件（共享，无冲突）；WakeWord 用 sherpa-onnx，
> RealtimeVoice 用 WebSocket/HTTP，**无原生库冲突**。两者都用 `Audio::FAudioCapture`
> 打开**默认麦克风**，停止时都会 `StopStream + CloseStream + Reset` 彻底释放设备，
> 因此"顺序交接"麦克风是干净的。

### 关键前置设置 / Required setup（两个坑）

1. **关掉 RealtimeVoice 的自动开始** — `RealtimeVoiceComponent.bAutoStartOnBeginPlay`
   **默认是 true**，会在 BeginPlay 抢麦。唤醒词驱动的流程里必须设为 **false**，
   让对话只在唤醒后才启动。WakeWord 的 `bAutoStartOnBeginPlay` 默认 false，把它设为 **true**
   （或手动 `StartListening`）当常驻监听者。
2. **启用 RealtimeVoice 插件** — 在 `.uproject` / `Edit → Plugins` 里同时启用 WakeWord 和 RealtimeVoice。

### 接线 / Wiring

```
唤醒 → 让出麦克风 → 开对话
RealtimeVoice.bAutoStartOnBeginPlay = false
WakeWord.bAutoStartOnBeginPlay      = true   (常驻监听)

WakeWord.OnWakeWordDetected ─▶ WakeWord.StopListening()      // 同步释放麦克风(会 join 工作线程)
                            ─▶ RealtimeVoice.StartSession()  // 连接后自动开麦(bAutoStartMicrophone=true)

对话结束 → 恢复监听
RealtimeVoice.OnSessionFinished ─▶ RealtimeVoice.StopSession()   // 关键: 保证麦克风被关
                                ─▶ WakeWord.StartListening()
```

为什么 `OnSessionFinished` 里要先 `StopSession()`：用户主动 `StopSession()` 结束时，
RealtimeVoice 会**先关麦**再断连，麦克风已释放；但若是**服务端**结束会话，`OnSessionFinished`
触发时麦克风可能仍开着。在回调里先调一次 `StopSession()` 可覆盖两种情况——它对重复调用是安全的
（已断连时只会再关一次麦，幂等），WakeWord 的 `StartListening()` 对"已在监听"也会安全跳过。

## 与 MetaHuman 口型 / With MetaHumanAudioToFace

唤醒本身不驱动口型；它只是"开关"。把 `On Wake Word Detected` 当作进入对话状态的触发器，
对话组件再把 TTS 音频流喂给 `MetaHumanAudioToFaceComponent` 做口型同步。

## 进阶：共用一个采集流 / Advanced: share one capture stream

如果你的对话管线也是用 `Audio::FAudioCapture` 在 UE 内采集，可以只打开一次设备，把回调
同时分发给 KWS 和对话两边，省一次设备打开/关闭。本插件默认各自独立打开麦克风（最简单、最稳）。
要共用时，建议在你自己的"音频路由"组件里持有 `FAudioCapture`，把 PCM 同时投递给两条管线。
