---
name: claude-code-unicode-audit
description: |
  Claude Code（Anthropic 的 AI 编程工具）针对中国用户的“标记”对人类肉眼不可见。本 skill 用于检查本机安装的 Claude Code 二进制，判断是否含有向系统提示词 “Today's date is” 注入近不可见 Unicode 撇号/分隔符的“隐写标记”代码，并在受影响时升级到干净版本。
metadata:
  version: "1.0.1"
  author: "wuwenbin-bj-st-01"
  license: "MIT"
---

# Claude Code 中国用户“标记”扫描器 V1.0.1

> **中文名**：Claude给中国用户的“标记”扫描器
> **作者**：wuwenbin-bj-st-01
> **版本**：1.0.1

Claude Code（Anthropic 的 AI 编程工具）针对中国用户的“标记”对人类肉眼不可见。本 skill 用于检查本机安装的 Claude Code 二进制，判断是否含有向系统提示词 “Today's date is” 注入近不可见 Unicode 撇号/分隔符的“隐写标记”代码，并在受影响时升级到干净版本。

## 何时使用
- 用户要求“检查/清理 Claude Code 系统提示词日期行的 Unicode 异常”
- 怀疑 Claude Code 在标记中国/代理用户（时区 Asia/Shanghai、Asia/Urumqi，或代理/中转域名）
- 定期安全自查、升级后复核

## 工作流程
1. 运行审计脚本，输出判定与证据：
   `python3 scripts/audit_cc_date_prompt.py`
   （加 `--json` 给机器可读结果；退出码 0=干净，2=受影响/可疑）
2. 读判定：
   - CLEAN：无隐写逻辑，结束。
   - SUSPECT：存在可疑字符串但非完整隐写信道，建议人工确认并升级。
   - AFFECTED：确认含隐写代码，执行清理。
3. 清理（仅 AFFECTED/SUSPECT，且需先经用户确认）：
   `python3 scripts/upgrade_cc.py --apply`
   该脚本会 `npm install -g @anthropic-ai/claude-code@latest` 并重新审计。
4. 重新运行步骤 1 确认已变为 CLEAN。

## 技术要点（给执行者）
- 可靠的检测目标是**二进制本身**：隐写注入逻辑编译在 `claude.exe` 内，运行时系统提示词难以直接 dump，故以二进制字符串/字节特征为准。
- 关键特征（详见 references/known_versions.md）：
  - 时区判定 `Asia/Shanghai` + `Asia/Urumqi`
  - 变量撇号模板 `Today${...}s date is`
  - XOR(密钥91) 域名/实验室关键词清单解码
  - 标记码点 U+2019 / U+02BC / U+02B9；命中时日期分隔符 `-`→`/`
- 受影响版本区间 v2.1.193–v2.1.196；v2.1.197 起回滚为干净版（latest 已含修复）。
- 仅升级能彻底去除（时区标记独立于代理，无法靠改配置绕过）。

## 注意事项
- 升级会改动全局 npm 包，属外部副作用，**必须先取得用户明确同意**再 `--apply`。
- 不要手动 patch 二进制（脆弱、易破坏、升级即覆盖）。
