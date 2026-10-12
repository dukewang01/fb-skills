---
name: proot-ubuntu-python
description: Use when Termux 装不上 manylinux 包时。proot Ubuntu 容器给真 glibc
version: 1
author: Hermes Agent
license: MIT
tags: [termux, proot, ubuntu, manylinux, python]
metadata:
  hermes:
    tags: [termux, proot, ubuntu, manylinux, python]
    related_skills: [android-termux-env, ocr-and-documents]
---

# proot Ubuntu 容器（给 Termux 补 glibc）

## When to Use

- Termux 原生 `pip install X` 报 `No matching distribution` 或
  `Failed to build 'pandas' when installing build dependencies`
- 需要 PyMuPDF / pandas / polars / kuzu / sqlite-vec / markitdown / docling 这类包
- **判断依据**：`python3 -c "import sysconfig;print(sysconfig.get_platform())"`
  返回 `android-*`（而不是 `linux-*`）→ 说明是 Bionic libc，manylinux 轮子全被拒

**不要用于**：Termux 原生能装的包（typst / pandoc / exiftool / graphviz 等）。
即时交互用的工具留在 Termux（快）；proot 有 20–50% ptrace 损耗。

## 为什么需要它

```
Termux  = Bionic libc，pip 只认 `android_*` 平台标签
  → PyPI 上的 manylinux_aarch64 轮子全部被拒 → 只能纯 python 轮或源码编译
真 glibc 的 Ubuntu → manylinux 轮子全部合法 → 一装就上
```

这是「换个能跑的发行版」，不是绕权限。

## 安装（Docker Hub 被墙时的走法）

`proot-distro install ubuntu` 默认从 `registry-1.docker.io` 拉，被墙时超时。
**改从国内镜像的 rootfs tarball 装**（install 支持 URL）：

```bash
pkg install -y proot-distro

BASE=https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/ubuntu-base/releases/24.04.4/release/ubuntu-base-24.04.4-base-arm64.tar.gz
proot-distro install -n ubuntu24 "$BASE"      # -n 自定义容器名
```

备选镜像：`mirrors.ustc.edu.cn/ubuntu-cdimage`、`mirrors.huaweicloud.com/ubuntu-cdimage`。
tarball 约 28.5 MB，展开约 500 MB。

## 首次配置

```bash
proot-distro login ubuntu24 -- /bin/bash -lc '
cat > /etc/apt/sources.list <<EOF
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ noble main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ noble-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ noble-security main restricted universe multiverse
EOF
rm -f /etc/apt/sources.list.d/*.sources
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y python3 python3-pip python3-venv ca-certificates
pip3 install --break-system-packages pymupdf pandas kuzu markitdown sqlite-vec
'
```

arm64 必须用 **`/ubuntu-ports/`**（不是 `/ubuntu/`）。`--break-system-packages` 绕 PEP 668。

## ⭐ 路径 / 挂载的两个坑

```
① **不要往 installed-rootfs/<name>/ 里写文件**
   proot-distro 5.x 的容器根**不是**那个目录的直读视图（写了不进容器 ✗ 白写）
② **容器里没有 /tmp 共享，也看不到 Termux home**
   能拿到的共享点只有：**/sdcard** 和 **/storage/emulated/0**
```

**正确做法：文件走 `/sdcard` 中转，拷入动作在容器内执行。**

```bash
# Termux 侧
cp ~/.hermes/fonts/NotoSansSC-Regular.otf /sdcard/Download/x/
# 容器内
proot-distro login ubuntu24 -- /bin/bash -lc '
mkdir -p /usr/share/fonts/truetype/hb && cp /sdcard/Download/x/* /usr/share/fonts/truetype/hb/
'
```

## ⭐⭐ PyMuPDF 写中文：`insert_text(fontfile=)` 会静默丢字

**症状**：写出 PDF 后读回，中文全变成 `·`，**但不报任何错**。

**根因**：容器 base image 里没有任何 CJK 字体，`fontfile=` 这条老 API 对
非 built-in 编码不做校验，字符被丢进 PDF 的不可映射槽位。

**先用 1 步备好字体**（容器里没有中文字体；字体要从 `/sdcard` 中转）：

```bash
cp ~/.hermes/fonts/NotoSansCJK-Regular.ttf /sdcard/Download/prootf/
proot-distro login ubuntu24 -- /bin/bash -lc '
mkdir -p /usr/share/fonts/truetype/hb && cp /sdcard/Download/prootf/*.ttf /usr/share/fonts/truetype/hb/'
```

用 **TTF**（`NotoSansCJK-Regular.ttf`），别用 SC 的 `.otf` 系列 —— 见下 CFF 注意。

```python
FONT = "/usr/share/fonts/truetype/hb/NotoSansCJK-Regular.ttf"

# ✗ 全部丢字（实测 CJK=0，读回是一串 ·）
page.insert_text((50,90), "中文", fontfile=FONT)

# ✓ 三种正确写法（2026-10-12 在容器内三种都实测 CJK 全留）
page.insert_font(fontname="hbsc", fontfile=FONT)          # ① 先注册
page.insert_text((50,90), "中文", fontname="hbsc")

page.insert_htmlbox(rect, '<div style="font-size:14px">中文</div>')   # ② 新版推荐

tw = pymupdf.TextWriter(page.rect)                        # ③ TextWriter
w.append((50,90), "中文", font=pymupdf.Font(fontfile=FONT), fontsize=13)
tw.write_text(page)
```

**验收必须「写→存→重开→读回」，并统计 CJK 字符数**，不能只看没有异常：

```python
CJK = lambda s: sum(1 for c in s if "\u4e00" <= c <= "\u9fff")
got = pymupdf.open("/tmp/t.pdf")[0].get_text().strip()
assert CJK(got) == CJK(原文), "中文丢了！"     # ⚠ 别用 len(got)
```

### 🚩 只数 `len(got)` = 假阳性（本机踩过）

`len(got)` 数的是**全部字符**，`···· 474· 88.1%` 也有长度。实测教训：安装日志打印
「✓ 中文字符数: 38」，回读值其实是 `'proot Ubuntu + PyMuPDF ···· 474· 88.1%'`
—— **CJK 真值 0**，脚本却因 `len()` 而判过，白报了一次成功。
**判据只能是 `\u4e00–\u9fff` 区间计数，且要拿原文长度做等式。**

### CFF/OTF 注意

`NotoSansSC-*.otf` 是 CFF 轮廓，PyMuPDF 走 `insert_font` 可能报
`code=4 cannot extract font file` 之类。要嵌就用 **TTF**（`NotoSansCJK-Regular.ttf`
或 `LXGWWenKai-*.ttf`）。

## 验证清单

- [ ] `sysconfig.get_platform()` 在容器里是 `linux-aarch64`（不是 `android-*`）
- [ ] apt 源用的是 `/ubuntu-ports/`（arm64）
- [ ] 目标包**真 import 成功**（不是看安装日志）
- [ ] 跑**一个真任务**（如写一个 PDF 再读回），并校验内容
- [ ] 中文类任务必须数 CJK 字符数（`\u4e00–\u9fff`），**不能用 `len()`**，且不能只看「没报错」

## 本机现状（2026-10-12 08:2x 复跑确认）

- 容器名 **`ubuntu24`**（Ubuntu 24.04.4 arm64 base，清华镜像 tarball 装的，约 500 MB）
- 容器内：`platform: linux-aarch64`；pymupdf 1.28.2 / pandas 3.0.6 / kuzu 0.11.3 /
  markitdown 0.1.8 / sqlite-vec 0.1.9 五个包**真 import 通过**
- 中文字体已装进容器 `/usr/share/fonts/truetype/hb/NotoSansCJK-Regular.ttf`
- 中文 PDF 三种写法（insert_font / insert_htmlbox / TextWriter）**读回 CJK 全留**，
  裸 `fontfile=` 路径复现丢字（CJK=0）——坑与正解均已实测
- **入口**：`proot-distro login ubuntu24 -- python3 /sdcard/xxx.py`
  （两边 Python 不相通，文件走 `/sdcard` 中转；容器看不到 Termux home）
