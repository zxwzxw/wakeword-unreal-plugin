# Testing & Verification / 测试与验证

## 自动化验证 / Automated verification (done at release)

1. **编译 + 链接 (5.6 与 5.7) / Compile + link (5.6 and 5.7)** — 5.6 编辑器目标
   `MyProject_bannEditor Win64 Development` 构建通过；5.7 经 `RunUAT BuildPlugin`（隔离 HostProject，
   含 `-WarningsAsErrors` 的 Game 目标）构建通过。两个引擎下 `WakeWordTokenizer/Component/Settings/Library/Module.cpp`
   均编译成功、`UnrealEditor-WakeWord.dll` 链接成功（验证 C-API 集成、delay-load 与 RuntimeDependencies 拷贝）。

2. **原生库 + 模型可用 / Runtime + model integrity** — 用插件**内置的**模型文件经官方
   `sherpa-onnx-keyword-spotter` 工具在模型自带 `test_wavs` 上检测全部命中：
   - 中文 `kws-zh`：`周望军, 朱丽楠, 蒋友伯, 文森特卡索, 落实, 见面会, 女儿, 法国`。
   - 英文 `kws-en`：模型自带 `keywords_raw.txt`（HELLO WORLD / HEY SIRI / ALEXA …）解码命中。

3. **自定义唤醒词正确性 / Custom-keyword correctness** — 两套分词器都与训练时分词**逐字一致**：
   - 中文：`text2token.py --lang zh --self-test Resources/kws-zh` 复现 pypinyin 分词 → **PASS**。
   - 英文：`text2token.py --lang en --bpe-model Resources/kws-en/bpe.model --self-test Resources/kws-en`
     复现 sentencepiece 分词 → **PASS**；运行时 C++ Viterbi 分词器读取派生的 `bpe-scores.txt`，
     在 9 对官方样本 + 28 条压力词上与 sentencepiece **完全一致**。

4. **keywords.txt 解析无误 / Keyword file parses cleanly** — 内置中/英 `keywords.txt`
   被 spotter 正常加载，且对无关语音不误触发。

## 手动在编辑器内测试 / Manual in-editor test (mic loopback)

自动化无法覆盖"真实麦克风采集 + 组件线程"这条链路，请按下面手测一次：

1. 关闭编辑器，按 README 的命令构建编辑器目标；重新打开工程。
2. 新建一个空 Actor 蓝图，添加 `Wake Word Component`，勾选 `Auto Start On Begin Play`。
3. 在 `On Wake Word Detected` 上接一个 `Print String`（打印 `Keyword`）。
4. 把蓝图拖进关卡，Play。
5. 对麦克风说唤醒词：
   - 中文（默认）：**"你好双蛙"**。
   - 英文：先在 项目设置→Wake Word 把 `Language` 切成 **English**，再说 **"HELLO FROG"**。
   屏幕应打印命中的短语。
6. 看 `Output Log` 的 `LogWakeWord`：应有 `Spotter ready ...` 与 `Detected: '...'`。

预期 / Expected：
- `On Listening Started` 在 Play 后很快触发。
- 正常说出唤醒词 → 1 秒内打印；冷却时间内不会重复刷屏。
- 误唤醒多 → 调高 `Keywords Threshold`；漏唤醒 → 调低或加 `:2.0` boosting。

## 排错 / Troubleshooting

| 现象 | 排查 |
|------|------|
| `On Error` code `-10` | 原生 DLL 没加载：确认 `Binaries/Win64` 下有 `sherpa-onnx-c-api.dll` / `onnxrt_ww.dll`（构建会自动拷）。 |
| `On Error` code `-11` | 模型文件缺失：用 `Get Model Dir` 看路径，确认 `Resources/kws-zh`（英文则 `kws-en`）下文件齐全。 |
| `On Error` code `-3`  | 麦克风打不开：检查默认录音设备与 Windows 麦克风权限。 |
| 一直无命中           | 确认在说 keywords.txt 里的词；适当调低阈值；`Num Threads` 设 1–2。 |
