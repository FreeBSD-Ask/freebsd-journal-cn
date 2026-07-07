# Tasks

- [x] Task 1: 创建图片下载脚本 `script/dl_2022_1112.py` 并下载 14 张 PNG 图片
  - [x] SubTask 1.1: 编写脚本，包含 14 个 GitHub user-attachments URL（act×1、huiyi×2、ipv6×6、prometheus×5）
  - [x] SubTask 1.2: 执行脚本下载至 `png/2022-1112/`，文件名按 `<slug>-NN.png` 命名
  - [x] SubTask 1.3: 验证 14 张图片全部下载成功且非空
- [x] Task 2: 下载 events-calendar PDF（原 URL 预期 404，需兜底）
  - [x] SubTask 2.1: 尝试原 URL `https://freebsdfoundation.org/wp-content/uploads/2023/01/events.pdf`（原 URL 实际可用，无需兜底）
  - [x] SubTask 2.2: 若 404，抓取 freebsdfoundation.org Nov/Dec 2022 期刊页面 HTML 查找正确 PDF URL（未触发，原 URL 成功）
  - [x] SubTask 2.3: 下载至 `png/2022-1112/events-calendar-01.pdf`（512046 bytes）
- [x] Task 3: 替换 `.md` 中远程图片引用为本地相对路径
  - [x] SubTask 3.1: act.md 行 18 → `../png/2022-1112/act-01.png`
  - [x] SubTask 3.2: events-calendar.md 行 10 → `../png/2022-1112/events-calendar-01.pdf`
  - [x] SubTask 3.3: huiyi.md 行 8/22 → `../png/2022-1112/huiyi-01.png`、`huiyi-02.png`
  - [x] SubTask 3.4: ipv6.md 行 59/68/241/245/251/255 → `../png/2022-1112/ipv6-01.png` 至 `ipv6-06.png`
  - [x] SubTask 3.5: prometheus.md 行 94/98/125/131/164 → `../png/2022-1112/prometheus-01.png` 至 `prometheus-05.png`
- [x] Task 4: 校正代码块内弯引号为直引号
  - [x] SubTask 4.1: act.md 行 96/98/113 — JSON 字符串弯引号 `'``'` → `'`
  - [x] SubTask 4.2: dtrace.md 行 41 — `printf("user = %u...` → `printf("user = %u...`
  - [x] SubTask 4.3: dtrace.md 行 256/257 — `printf("%s: user = %u...` → `printf("%s: user = %u...`
  - [x] SubTask 4.4: icinga.md 行 120 — sed 命令行尾弯引号 `'` → `'`
  - [x] SubTask 4.5: icinga.md 行 283 — `'1.14.3'` → `'1.14.3'`
- [x] Task 5: 验证 ddb.md 行 207/211 中文注释保留全角不动
- [x] Task 6: 最终验证
  - [x] SubTask 6.1: Grep 确认 2022-1112/ 下无远程 `![...](http` 引用残留（0 匹配）
  - [x] SubTask 6.2: Grep 确认代码块内无弯引号残留（排除中文注释 ddb.md 207/211）
  - [x] SubTask 6.3: 确认 `png/2022-1112/` 下有 14 PNG + 1 PDF 共 15 个文件

# Task Dependencies

- Task 2 独立于 Task 1，可并行
- Task 3 依赖 Task 1 + Task 2 完成
- Task 4 独立于图片任务，可并行
- Task 5 独立验证
- Task 6 依赖 Task 3 + Task 4 + Task 5
