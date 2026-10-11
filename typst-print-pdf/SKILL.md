---
name: typst-print-pdf
description: Use when 要出印刷品 PDF(海报/报告/A3/A4)。Typst 一行出精确纸张尺寸
version: 1
author: Hermes Agent
license: MIT
tags: [pdf, typst, a3, poster, typesetting, cjk]
metadata:
  hermes:
    tags: [pdf, typst, a3, poster, typesetting, cjk]
    related_skills: [single-file-html-editing, hand-drawn-poster]
---

# Typst 印刷品 PDF

用 Typst 出**纸张尺寸精确**的 PDF（海报 / 报告 / 手册 / 说明卡）。
**替代**「HTML → headless Chromium 截图 → 图片/PDF 拼装」那条三段式链路。

## When to Use

- 要出 **A3 / A4 / 自定义尺寸**的印刷品，且尺寸必须精准
- 要**反复改版**（改一行重编译即可）
- 纯排版内容：标题 / 方框 / 表格 / 分栏 / 公式
- 需要**内嵌中文字体**、体积要小（0.07 MB 级）

**不要用于**：
- 交互式单文件 HTML（折叠 / 搜索 / 勾选）→ 走 `single-file-html-editing`
- 复杂图表（ECharts / 力导向 / 动态）→ 走 HTML + Chromium 截图
- 一句话结论：**Typst 管印刷品，HTML 管交互品**

## 环境（本机已装，2026-10-12 实测）

```bash
typst --version    # typst 0.15.1
```

### ⭐ 中文字体（两件事都要做，少一件就没字或报警告）

```bash
# 1) 字体家族名是 "Noto Sans CJK JP" —— 不是 SC！
typst fonts                     # 只看得到 4 个内置西文字体
# 2) 必须用 --font-path 显式指过去（typst 不走 fontconfig）
typst compile --font-path ~/.hermes/fonts a.typ a.pdf
```

```typst
#set text(font: "Noto Sans CJK JP", size: 15pt)
```

> 写 `"Noto Sans CJK SC"` 只会得到 `warning: unknown font family`，
> 虽然靠回退仍能出字，但字体匹配不确定 —— **写 JP 那个真名，得到零警告**。

## 命令

```bash
typst compile --font-path ~/.hermes/fonts in.typ out.pdf          # PDF
typst compile --font-path ~/.hermes/fonts --format png --ppi 150 in.typ out.png
typst watch  --font-path ~/.hermes/fonts in.typ out.pdf           # 改存自动重编
```

验证产物：MediaBox 换算成 mm 应等于目标纸张。

```python
import re
from PIL import Image
s = open('out.pdf','rb').read().decode('latin1')
mb = [float(x) for x in re.search(r'/MediaBox\s*\[([^\]]+)\]', s).group(1).split()]
print('%.0f x %.0f mm' % ((mb[2]-mb[0])*25.4/72, (mb[3]-mb[1])*25.4/72))  # A3 = 297 x 420
```

## 可复用模板骨架

```typst
#set page(paper: "a3", margin: (x: 26mm, y: 22mm), fill: white)
#set text(font: "Noto Sans CJK JP", size: 15pt, fill: rgb("#1f2a37"))

#let ink  = rgb("#1f2a37")
#let soft = rgb("#f4f7fa")
#let line = rgb("#b8c6d4")
#let acc  = rgb("#2f5d8f")

// ⚠️ stroke 必须是字典，不能写 `长度 + 颜色`
#let node(w: 100%, fill: white, st: 1.2pt, pt: line, pad: 11pt, b) = box(
  width: w, fill: fill, stroke: (paint: pt, thickness: st), radius: 5pt,
  inset: (x: 12pt, y: pad), align(center + horizon)[#b]
)
#let dark(b) = box(
  width: 100%, fill: ink, radius: 6pt,
  inset: (x: 14pt, y: 15pt), align(center + horizon)[#text(fill: white)[#b]]
)
#let arrow() = align(center)[#text(size: 17pt, fill: acc)[↓]]

#align(center)[#text(size: 25pt, weight: "bold")[主标题]]
#v(16pt)
#dark[*顶层框* #h(6pt) ｜ #h(6pt) 副信息]
#v(5pt)
#arrow()
#grid(columns: (1fr, 1fr, 1fr), column-gutter: 12pt,
  node(fill: soft)[*左*], node(fill: soft)[*中*], node(fill: soft)[*右*],
)
```

常用容器：`box`/`block`（方框）、`grid`（等高分栏，比 columns 更适合卡片）、
`stack(dir: ttb)`、`place`（绝对定位）、`rect`/`line`（画形状）、
`align(center + horizon)`（水平垂直居中）。

## Pitfalls

### 1. stroke 不能直接写「长度 + 颜色」

```typst
stroke: 1.2pt + line        // ✗ error: cannot add stroke and color
stroke: (paint: line, thickness: 1.2pt)   // ✓
```

### 2. ⭐ 不要把聊天用的分隔符写进正式文档

聊天里常用 `✗` 当分隔符（口语化习惯），**但正式文档里 `✗` 是「错误 / 否定」**。

```
误：养老金优先保 ✗ 只涨不降     → 被读成「养老金优先保是错的」
正：养老金优先保 ｜ 只涨不降
```

同理 `✓` `⚠️` `＋` 都要按**读者视角**复核一遍。
写完后 grep 成品正文里的这类符号，**留下的是语义标记，不是分隔符**。

### 3. ⭐ 编译失败绝不能覆盖旧产物

Typst 编译失败时**不写输出文件**（旧文件留在原地）。
若脚本先 `cp` 再编译、或不检查退出码，就会**把上一版冒充成新版**。

```bash
rm -f out.pdf out.png                      # 先删，防拿旧的冒充
typst compile ... || { echo '编译失败，不覆盖'; exit 1; }
cp out.pdf "$DEST/"                        # 成功才拷
```

### 4. `--font-path` 不给就回落西文字体

不写 `--font-path` 时 typst 只认识 4 个内置字体，中文靠不确定的回退，
**表现为「能出字但一直有 unknown font family 警告」**。

### 5. 换行 / 强制断行

Typst 是标记语言：`\` 断行，空行分段；`[...]` 内是内容块，
`#` 开头才是代码。**块内调用函数要带 `#`**。

## 自检清单（发出去之前）

- [ ] MediaBox 换算 mm == 目标纸张（A3 = 297 × 420）
- [ ] 编译**零 warning**（有 `unknown font family` 就是字体名写错）
- [ ] PDF 里 `FontFile` 存在（字体真内嵌，换设备不掉字）
- [ ] 渲染 PNG 后**分区读图**复核：中文无方框 / 无溢出 / 无重叠 / 无裁切
- [ ] 正文里没有把 `✗ ✓ ⚠️` 当分隔符用
- [ ] 产物已归位，且是**本次编译**的结果（md5 与上一版不同）
