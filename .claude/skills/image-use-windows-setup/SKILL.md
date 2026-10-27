---
name: image-use-windows-setup
version: 1.0.0
display_name: image-use Windows 环境配置
display_name_en: image-use Windows Setup
description: >-
  Configure or repair the image-use ChatGPT web-generation channel on Windows
  (Git Bash/MSYS): install chrome-use via install.ps1 through a local proxy,
  register the native-messaging host, install the Chrome Web Store extension,
  persist PYTHONUTF8=1, and verify with a real test generation. Use when the
  user asks to set up, install, or fix image generation on Windows, or when
  image-use doctor reports chrome-use missing, a run dies with GBK
  UnicodeDecodeError, or the composer-leftover-draft error appears.
description_zh: >-
  在 Windows Git Bash 环境配置或修复 image-use 的 ChatGPT 生图渠道：经本地代理用
  install.ps1 安装 chrome-use，注册 Native Messaging Host，安装 Chrome 商店扩展，
  持久化 PYTHONUTF8=1，并用真实出图验证。当用户要求配置、安装或修复 Windows 生图环境，
  或 image-use doctor 报 chrome-use 缺失，或生图报 GBK 编码错误、
  ChatGPT 输入框残留草稿时使用。
description_en: >-
  Configure or repair the image-use ChatGPT web-generation channel on Windows
  (Git Bash/MSYS): install chrome-use via install.ps1 through a local proxy,
  register the native-messaging host, install the Chrome Web Store extension,
  persist PYTHONUTF8=1, and verify with a real test generation. Use when the
  user asks to set up, install, or fix image generation on Windows, or when
  image-use doctor reports chrome-use missing, a run dies with GBK decode
  errors, or the composer-leftover-draft error appears.
---

# image-use Windows 环境配置

在 Windows（Git Bash/MSYS）上打通 image-use 的 ChatGPT `web` 生图渠道：
chrome-use CLI → Native Messaging Host → Chrome 商店扩展 → 编码修复 → 真实出图验证。

本流程在 Windows 10/11 + Git Bash + 中文区域（GBK）机器上完整验证过（2026-02，chrome-use 1.5.268 / image-use 0.34.5）。

## 何时使用

- 用户要求"配置/安装/修复 image-use 生图环境"（Windows）
- `image-use doctor` 报 `chrome-use not installed` 或 `relay not connected`
- 生图报错 `UnicodeDecodeError: 'gbk' codec` 或 `ChatGPT composer is not empty`

## 边界

- 只负责 **web 渠道**（ChatGPT 网页出图，不消耗 Codex 额度）。**codex 路线（路线 B）不配置**——保持 `codex login` 未执行即等效禁用，auto 模式不会误降级。
- 不负责生图本身的提示词编写、`animate` 动图依赖（magick/img2webp）的安装。
- Linux/macOS 环境走官方 `install.sh`，不适用本技能。

## 输入

- 本地代理端口（没有则无法下载 GitHub 资源，需先问用户或探测）
- image-use 仓库位置（本机为 `E:/DesignProjects/image-use`）

## 执行

进入 [references/setup-flow.md](references/setup-flow.md)，按阶段顺序执行：
每个阶段末尾有验证点，通过才进入下一阶段；坑点以 `坑N` 标注，直接给出已验证的修法。

## 完成标准

1. `chrome-use extension status` 显示 `live extension version` 和 driving profile
2. `image-use doctor` 的 web backend 两项 `[ok]`（`profiles none detected` 在 Windows 上可忽略，见坑5）
3. 一次真实出图成功保存 PNG
