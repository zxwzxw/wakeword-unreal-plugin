# Third-Party Notices

The **WakeWord** plugin redistributes the following third-party components. Each is
the property of its respective owners and is licensed under the terms below. These
notices are provided to comply with the corresponding licenses; they do not modify
the license that governs the WakeWord plugin itself.

---

## 1. sherpa-onnx (k2-fsa)

- Project: https://github.com/k2-fsa/sherpa-onnx
- Version: v1.13.2 (Windows x64, shared, MD runtime)
- Files: `Source/ThirdParty/SherpaOnnx/**`, `Binaries/Win64/sherpa-onnx-c-api.dll`
- License: **Apache License 2.0**

```
                                 Apache License
                           Version 2.0, January 2004
                        http://www.apache.org/licenses/

   Copyright 2022-2024  Xiaomi Corporation (authors: Fangjun Kuang, et al.)
                        and the k2-fsa / sherpa-onnx contributors.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
```

The full Apache-2.0 license text is available at
<http://www.apache.org/licenses/LICENSE-2.0>.

---

## 2. ONNX Runtime (Microsoft)

- Project: https://github.com/microsoft/onnxruntime
- File: `Binaries/Win64/onnxrt_ww.dll` (ONNX Runtime v1.24, redistributed as bundled with
  the sherpa-onnx release). It is shipped under this **private file name** (and
  `sherpa-onnx-c-api.dll`'s import table was patched accordingly via
  `Tools/patch_onnxruntime_name.py`) to avoid a base-name collision with the
  differently-versioned `onnxruntime.dll` that Unreal Engine's NNE plugin and
  `C:\Windows\System32` load into the same process. The binary is otherwise unmodified.
- License: **MIT License**

```
MIT License

Copyright (c) Microsoft Corporation. All rights reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 3. Keyword-spotting model — sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01

- Source: https://www.modelscope.cn/pkufool/sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01
  (mirror: https://github.com/k2-fsa/sherpa-onnx/releases/tag/kws-models)
- Files: `Resources/kws-zh/*.onnx`, `Resources/kws-zh/tokens.txt`
- Training data: WenetSpeech (L subset, ~10000 h); modelling unit: pinyin (声母 + 韵母).
- License: **Apache License 2.0** (see section 1 for the license text).

---

## 4. Keyword-spotting model — sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01

- Source: https://www.modelscope.cn/pkufool/sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01
  (mirror: https://github.com/k2-fsa/sherpa-onnx/releases/tag/kws-models)
- Files: `Resources/kws-en/*.onnx`, `Resources/kws-en/tokens.txt`, `Resources/kws-en/bpe.model`.
  `Resources/kws-en/bpe-scores.txt` is derived from `bpe.model` (the sentencepiece piece/score
  table, extracted to drive the in-engine tokenizer) and is covered by the same license.
- Training data: GigaSpeech (XL subset, ~10000 h); modelling unit: sentencepiece BPE/unigram.
- License: **Apache License 2.0** (see section 1 for the license text).

---

## 5. pypinyin (used by Tools/text2token.py only — NOT redistributed)

- Project: https://github.com/mozillazg/python-pinyin
- License: MIT License
- Note: `Tools/text2token.py` imports `pypinyin` at authoring time to generate Chinese keyword
  token sequences. It is a developer tool and is not required at runtime; the pypinyin
  package itself is not bundled with this plugin.

---

## 6. sentencepiece (used by Tools/text2token.py only — NOT redistributed)

- Project: https://github.com/google/sentencepiece
- License: Apache License 2.0
- Note: `Tools/text2token.py --lang en` imports `sentencepiece` at authoring time to generate /
  verify English keyword token sequences from `bpe.model`. It is a developer tool and is not
  required at runtime (the plugin reproduces the same tokenization in C++ from `bpe-scores.txt`);
  the sentencepiece library itself is not bundled with this plugin.
