# WakeWord Wiki

Offline wake-word / keyword spotting for Unreal Engine 5.6 / 5.7 (sherpa-onnx, on-device).
Bundles **Chinese (wenetspeech)** and **English (gigaspeech)** 3.3M models, switchable by language.

## Pages
- 🇨🇳 [中文教程](GitHub-Wiki.zh-CN) — 安装、上手、面板/蓝图管理唤醒词、调参、对话管线接入、API、排错、FAQ
- 🇬🇧 [English Guide](GitHub-Wiki.en) — install, quick start, managing wake words, tuning, dialog pipeline, API, troubleshooting, FAQ

## How to publish these to a GitHub wiki
1. Enable the **Wiki** tab on the GitHub repo.
2. Clone the wiki repo: `git clone https://github.com/<you>/<repo>.wiki.git`
3. Copy these files in, renaming to the wiki page names:
   - `GitHub-Wiki.Home.md`  → `Home.md`
   - `GitHub-Wiki.zh-CN.md` → `中文教程.md` (or `GitHub-Wiki.zh-CN.md`)
   - `GitHub-Wiki.en.md`    → `English-Guide.md` (or `GitHub-Wiki.en.md`)
4. `git add . && git commit -m "WakeWord wiki" && git push`

> Keep page-name links in sync if you rename the files.

## Quick links
- Blueprint usage: [Docs/BlueprintGuide.md](BlueprintGuide.md)
- C++ usage: [Docs/CppGuide.md](CppGuide.md)
- Dialog pipeline: [Docs/Integration.md](Integration.md)
- Testing: [Docs/Testing.md](Testing.md)
