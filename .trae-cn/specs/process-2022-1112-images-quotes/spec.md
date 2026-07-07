# 2022-1112 期图片离线与代码块引号校正 Spec

## Why

2022 年 11/12 月期刊（2022-1112 期）的 11 篇 `.md` 文件中，仍有 15 处远程图片引用（GitHub user-attachments 与 freebsdfoundation.org PDF）未下载到本地 `png/2022-1112/` 目录，且代码块内存在弯引号 `'’""` 需要校正为直引号。本期是 2022 年 6 期期刊处理的最后一批，前 5 期（0102、0304、0506、0708、0910）均已完成。

## What Changes

- **图片离线（阶段 P）**：下载 14 张 PNG 图片（act×1、huiyi×2、ipv6×6、prometheus×5）至 `png/2022-1112/`，并将各 `.md` 中的远程 GitHub user-attachments URL 替换为本地相对路径 `../png/2022-1112/<文件名>.png`
- **PDF 兜底**：下载 events-calendar.md 中的 1 个 PDF（原 URL `https://freebsdfoundation.org/wp-content/uploads/2023/01/events.pdf` 预期 404），通过抓取 freebsdfoundation.org Nov/Dec 2022 期刊页面 HTML 查找正确 URL 后下载至 `png/2022-1112/events-calendar-01.pdf`
- **代码块引号校正（阶段 Q）**：校正以下代码块内的弯引号为直引号：
  - `act.md` 行 96/98/113：shell 脚本中 JSON 字符串的弯引号 `'``'`
  - `dtrace.md` 行 41/256/257：dtrace 命令字符串内 `printf("...")` 的弯引号 `""`
  - `icinga.md` 行 120：sed 命令行尾弯引号 `'`
  - `icinga.md` 行 283：命令输出中 `'1.14.3'` 的弯引号
- **保留全角**：`ddb.md` 行 207/211 是 C 代码块中的中文注释（`/* 保存"demo *"命令的列表。*/`），保留全角弯引号不动

## Impact

- 受影响文件：`2022-1112/act.md`、`2022-1112/events-calendar.md`、`2022-1112/huiyi.md`、`2022-1112/ipv6.md`、`2022-1112/prometheus.md`、`2022-1112/dtrace.md`、`2022-1112/icinga.md`
- 新增文件：`png/2022-1112/` 下 14 张 PNG + 1 个 PDF
- 临时脚本：`script/dl_2022_1112.py`（保留不清理，遵循用户规则）
- 不影响：`# 标题`（被 CI 覆盖）、代码语义、版本号、用户名、带圈数字、URL/IP

## ADDED Requirements

### Requirement: 图片本地化

系统 SHALL 将 2022-1112 期所有 `.md` 文件中的远程图片引用下载至 `png/2022-1112/`，并替换为本地相对路径 `../png/2022-1112/<文件名>`。

#### Scenario: 图片下载成功

- **WHEN** 远程图片 URL 可访问
- **THEN** 下载到 `png/2022-1112/<文件名>.png` 并替换 `.md` 中引用为 `../png/2022-1112/<文件名>.png`

#### Scenario: PDF URL 404 兜底

- **WHEN** events-calendar PDF 原 URL 返回 404
- **THEN** 通过抓取 freebsdfoundation.org 期刊目录页 HTML 查找正确 PDF URL 后下载

### Requirement: 代码块引号校正

系统 SHALL 将围栏代码块内非注释部分、行内代码内部的弯引号 `'’""` 校正为直引号 `'"`。

#### Scenario: 命令字符串内弯引号

- **WHEN** 代码块内命令字符串（如 `printf("...")`、`sed 's/.../.../'`）含弯引号
- **THEN** 替换为直引号，不改变命令语义

#### Scenario: 中文注释内弯引号

- **WHEN** 代码块内中文注释（如 `/* 中文注释 */`）含全角弯引号
- **THEN** 保留全角不动

## MODIFIED Requirements

无（本期为新增处理，不修改既有规范）。

## REMOVED Requirements

无。
