# C++ Guide / C++ 使用

在你的模块 `Build.cs` 里依赖 `WakeWord`：

```csharp
PrivateDependencyModuleNames.AddRange(new[] { "WakeWord" });
```

绑定事件并启动监听：

```cpp
#include "WakeWordComponent.h"

void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    WakeWord = NewObject<UWakeWordComponent>(this);
    WakeWord->RegisterComponent();

    // 可选调参 / optional tuning
    WakeWord->KeywordsThreshold = 0.25f;
    WakeWord->RetriggerCooldownSeconds = 1.0f;

    WakeWord->OnWakeWordDetected.AddDynamic(this, &AMyActor::HandleWakeWord);
    WakeWord->OnError.AddDynamic(this, &AMyActor::HandleWakeWordError);

    if (WakeWord->IsReady())
    {
        WakeWord->StartListening();
    }
}

void AMyActor::HandleWakeWord(const FString& Keyword)
{
    UE_LOG(LogTemp, Log, TEXT("Wake word: %s"), *Keyword);
    WakeWord->StopListening();      // 让出麦克风给对话管线 / free the mic
    // ... 启动你的对话会话 ...
}

void AMyActor::HandleWakeWordError(int32 Code, const FString& Message)
{
    UE_LOG(LogTemp, Warning, TEXT("WakeWord error %d: %s"), Code, *Message);
}
```

`UFUNCTION` 回调签名 / matching UFUNCTION signatures:

```cpp
UFUNCTION() void HandleWakeWord(const FString& Keyword);
UFUNCTION() void HandleWakeWordError(int32 Code, const FString& Message);
```

### 中文 / 英文 切换 / Choosing the language

唤醒词语言由项目设置 `Language` 决定，组件 `Model.Preset` 默认 `Auto` 跟随它；也可在组件上强制：

```cpp
#include "WakeWordLibrary.h"
#include "WakeWordSettings.h"   // EWakeWordLanguage

// 全局切到英文模型(并持久化)，组件 Preset=Auto 即自动跟随
UWakeWordLibrary::SetLanguage(EWakeWordLanguage::English, /*bSave=*/true);
UWakeWordLibrary::AddWakeWord(TEXT("OPEN THE DOOR"));   // 英文原文, 内部自动 BPE 分词

// 或在组件上强制语言, 不依赖项目设置:
WakeWord->Model.Preset = EWakeWordModelPreset::English;   // 或 ::Chinese / ::Auto / ::Custom
WakeWord->RefreshWakeWords();
```

运行时换词（高级，原始 token）/ Swap keywords at runtime (advanced, raw tokens)：

```cpp
// 中文(拼音 token) / Chinese (pinyin tokens)
TArray<FString> Zh = { TEXT("n ǐ h ǎo sh uāng w ā @你好双蛙") };
WakeWord->SetCustomKeywords(Zh, /*bRestartIfListening=*/true);

// 英文(BPE token, ▁ 是词首标记) / English (BPE tokens, ▁ = word-start)
TArray<FString> En = { TEXT("▁HE LL O ▁F RO G @HELLO FROG") };
WakeWord->SetCustomKeywords(En, /*bRestartIfListening=*/true);
```

### 线程模型 / Threading

- 麦克风在 GameThread 开关；音频回调在设备线程做降混 + 重采样 + 入队（SPSC）。
- 工作线程消费队列、跑 sherpa-onnx 解码，命中后 `AsyncTask(GameThread)` 广播。
- 所有委托都在 **GameThread** 触发，回调里可安全操作 UObject / UI。
