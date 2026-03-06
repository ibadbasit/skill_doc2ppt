---
name: ppt
description: 将 .md 或 .docx 文件转换为可投屏的单文件 HTML 演示文稿。设计风格：暖奶油×赭红×深棕黑，Cormorant Garamond + Noto Serif SC，键盘翻页+导航点。用法：/ppt <文件路径>
tools: Read, Write, Bash
---

# 演示文稿生成器

将 `$ARGUMENTS` 转换为演示文稿。

## 参数解析（首先执行）

检查 `$ARGUMENTS` 是否包含 `--pptx`：

- **包含 `--pptx`**：提取文件路径（去掉 `--pptx` 标志），走 **PPTX 分支**（见文件末尾）
- **不含 `--pptx`**：走现有 **HTML 分支**（第一步至第三步），逻辑不变

**HTML 输出路径**：将输入文件的扩展名替换为 `.html`，保存到同目录。
例：`/path/to/report.md` → `/path/to/report.html`

**PPTX 输出路径**：将输入文件的扩展名替换为 `.pptx`，保存到同目录。
例：`/path/to/report.md --pptx` → `/path/to/report.pptx`

---

## 第一步：读取源文件

如果是 `.docx`，用以下方式解包提取文本和图片：

```python
import zipfile, os
with zipfile.ZipFile('input.docx') as z:
    imgs = [f for f in z.namelist() if f.startswith('word/media/')]
    z.extractall('/tmp/docx_extracted/')
# 解析 word/document.xml 获取正文
# 映射 word/_rels/document.xml.rels 获取图片与段落的对应关系
```

图片以 base64 嵌入 HTML，不依赖外部路径。

如果是 `.md`，直接读取文本内容。

---

## 第二步：规划幻灯片

**内容原则（最重要，必须严格遵守）：**

- 只用原文里有的内容，绝不补充、解释、扩写
- 找到每个段落最重的那一句，让它单独站出来——那句话就是这张幻灯片
- 保留作者的语言，包括口语、俚语、反常识的表达
- 删除一切可以用于任何行业/任何报告的套话
- 每张幻灯片只说一件事，说完就停，不解释，不总结

**幻灯片类型选择：**

| 类型 | 背景 | 用途 |
|------|------|------|
| `dark` | `#231510` 深棕黑 | 核心判断、金句，一句话站满一张 |
| `terra` | `#B05A42` 赭红 | 章节分隔页（用大编号 01/02/03） |
| `cream` | `#F4EDE3` 暖奶油 | 正文、数据、表格、列表 |

**节奏规则：**
- 章节页（terra）→ 若干内容页（cream/dark）→ 下一章节
- dark 页不连续超过 2 张
- 封面用 dark，结尾用 cream

**典型结构：**
```
封面（dark）      标题 + 副标题 + 日期/来源
章节页（terra）   大编号 + 章节名
内容页（cream）   caption → 40px 分隔线 → 主体内容
核心判断页（dark）单句 pull-light，字号放大，留白充足
图表页（cream）   左：文字判断 / 右：图表（base64）
结尾页（cream）   收尾信息
```

---

## 第三步：生成 HTML

使用以下完整模板，填入实际幻灯片内容：

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>演示文稿标题</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,600;1,300;1,400&family=Noto+Serif+SC:wght@300;400;600&display=swap" rel="stylesheet">
<style>
:root {
  --cream:     #F4EDE3;
  --terra:     #B05A42;
  --terra-dim: #C4795F;
  --ink:       #231510;
  --ink-soft:  #5C3D30;
  --ink-faint: #9E7B6E;
  --rule:      #D4BFB0;
}
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  background: #000;
  font-family: 'Cormorant Garamond', 'Noto Serif SC', serif;
  overflow: hidden;
  width: 100vw; height: 100vh;
}
.presentation { width: 100vw; height: 100vh; position: relative; }
.slide {
  position: absolute; inset: 0;
  display: flex; flex-direction: column;
  justify-content: center; align-items: flex-start;
  padding: clamp(2.5rem, 7vw, 5rem) clamp(3rem, 10vw, 7rem);
  opacity: 0; pointer-events: none;
  transition: opacity 0.45s ease;
}
.slide.active { opacity: 1; pointer-events: all; }
.slide.dark  { background: var(--ink);   color: var(--cream); }
.slide.terra { background: var(--terra); color: var(--cream); }
.slide.cream { background: var(--cream); color: var(--ink-soft); }

/* 章节页大数字 */
.display-num {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(5rem, 15vw, 12rem);
  font-weight: 300; line-height: 0.85;
  color: rgba(255,255,255,0.2);
}
.chapter-title {
  font-size: clamp(1.8rem, 3.5vw, 2.8rem);
  font-weight: 300; color: var(--cream); margin-top: 1.5rem;
}

/* 封面 */
.cover-title {
  font-size: clamp(2rem, 4.5vw, 3.8rem);
  font-weight: 300; color: var(--cream);
  line-height: 1.35; max-width: 75%;
}
.cover-meta {
  font-size: clamp(0.75rem, 1.1vw, 0.95rem);
  color: rgba(244,237,227,0.4);
  margin-top: 2.5rem; letter-spacing: 0.08em; line-height: 1.9;
}

/* Pull quotes */
.pull-light {
  font-size: clamp(1.6rem, 3.2vw, 2.6rem);
  font-weight: 300; line-height: 1.55;
  color: var(--cream); max-width: 80%;
}
.pull-quote {
  font-size: clamp(1.4rem, 2.8vw, 2.2rem);
  font-weight: 300; line-height: 1.55;
  color: var(--ink-soft); max-width: 70%;
}

/* Caption + 分隔线 */
.caption {
  font-size: clamp(0.62rem, 0.85vw, 0.72rem);
  letter-spacing: 0.14em; text-transform: uppercase;
  color: var(--ink-faint); margin-bottom: 1.4rem;
}
.caption-light {
  font-size: clamp(0.62rem, 0.85vw, 0.72rem);
  letter-spacing: 0.14em; text-transform: uppercase;
  color: rgba(244,237,227,0.4); margin-bottom: 1.4rem;
}
.rule       { width: 40px; height: 1px; background: var(--terra);     margin-bottom: 2rem; }
.rule-light { width: 40px; height: 1px; background: var(--terra-dim); margin-bottom: 2rem; }

/* 正文 */
.body-text {
  font-size: clamp(0.9rem, 1.5vw, 1.15rem);
  line-height: 1.95; color: var(--ink-soft); max-width: 62ch;
}
.body-text-light {
  font-size: clamp(0.9rem, 1.5vw, 1.15rem);
  line-height: 1.95; color: rgba(244,237,227,0.6);
  max-width: 62ch; margin-top: 1.5rem;
}

/* 表格 */
.data-table { border-collapse: collapse; width: 100%; max-width: 640px; margin-top: 0.5rem; }
.data-table th {
  font-size: clamp(0.6rem, 0.82vw, 0.7rem); letter-spacing: 0.12em;
  text-transform: uppercase; color: var(--ink-faint);
  padding: 0.5rem 2rem 0.5rem 0; text-align: left;
  font-weight: 400; border-bottom: 1px solid var(--rule);
}
.data-table td {
  font-size: clamp(0.85rem, 1.3vw, 1rem); color: var(--ink-soft);
  padding: 0.7rem 2rem 0.7rem 0; border-bottom: 1px solid var(--rule);
  line-height: 1.5;
}

/* 大数字统计 */
.big-stat {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(4rem, 11vw, 8rem); font-weight: 300; line-height: 1;
}
.stat-label { font-size: clamp(0.8rem, 1.2vw, 1rem); color: var(--ink-faint); margin-top: 0.4rem; }

/* 图表页（左文右图） */
.chart-slide { flex-direction: row !important; align-items: stretch !important; padding: 0 !important; }
.chart-left  { flex: 1; display: flex; flex-direction: column; justify-content: center;
               padding: clamp(2.5rem, 7vw, 5rem) clamp(2rem, 4vw, 3.5rem); }
.chart-right { flex: 1; display: flex; align-items: center; justify-content: center;
               background: #1a0d08; padding: 2rem; }
.chart-right img { max-width: 100%; max-height: 80vh; object-fit: contain; }

/* 进度条 */
#progress-bar {
  position: fixed; top: 0; left: 0; height: 2px;
  background: var(--terra); transition: width 0.35s ease; z-index: 100;
}
/* 导航点 */
#nav-dots {
  position: fixed; bottom: 1.5rem; left: 50%; transform: translateX(-50%);
  display: flex; gap: 6px; z-index: 100;
}
.dot {
  width: 5px; height: 5px; border-radius: 50%;
  background: rgba(156,123,110,0.3); cursor: pointer;
  transition: background 0.3s, transform 0.2s;
}
.dot:hover { transform: scale(1.4); }
.dot.active { background: var(--terra); }
/* 页码 */
#slide-counter {
  position: fixed; bottom: 1.4rem; right: 2rem;
  font-family: 'Cormorant Garamond', serif;
  font-size: 0.75rem; letter-spacing: 0.1em; color: var(--ink-faint); z-index: 100;
}
</style>
</head>
<body>

<div id="progress-bar"></div>
<div class="presentation">

  <!-- 在此填入幻灯片 -->

  <!-- 封面示例 -->
  <div class="slide dark active">
    <div class="caption-light">副标题或来源</div>
    <div class="cover-title">主标题</div>
    <div class="cover-meta">日期 · 数据来源</div>
  </div>

  <!-- 章节页示例 -->
  <div class="slide terra">
    <div class="display-num">01</div>
    <div class="chapter-title">章节名称</div>
  </div>

  <!-- 内容页示例（cream）-->
  <div class="slide cream">
    <div class="caption">小标题 CAPTION</div>
    <div class="rule"></div>
    <div class="body-text">正文内容……</div>
  </div>

  <!-- 核心判断页示例（dark）-->
  <div class="slide dark">
    <div class="caption-light">可选 caption</div>
    <div class="rule-light"></div>
    <div class="pull-light">这里是最重的那一句判断。</div>
  </div>

  <!-- 图表页示例（chart）-->
  <div class="slide cream chart-slide">
    <div class="chart-left">
      <div class="caption">图表说明</div>
      <div class="rule"></div>
      <div class="pull-quote">左侧放对图表的判断文字</div>
    </div>
    <div class="chart-right">
      <img src="data:image/png;base64,..." alt="图表">
    </div>
  </div>

</div><!-- .presentation -->

<div id="nav-dots"></div>
<div id="slide-counter"></div>

<script>
const slides = document.querySelectorAll('.slide');
const progressBar = document.getElementById('progress-bar');
const navDots = document.getElementById('nav-dots');
const slideCounter = document.getElementById('slide-counter');
let cur = 0;

slides.forEach((_, i) => {
  const dot = document.createElement('div');
  dot.className = 'dot' + (i === 0 ? ' active' : '');
  dot.addEventListener('click', () => go(i));
  navDots.appendChild(dot);
});

function updateUI() {
  progressBar.style.width = ((cur + 1) / slides.length * 100) + '%';
  document.querySelectorAll('.dot').forEach((d, i) => d.classList.toggle('active', i === cur));
  slideCounter.textContent = (cur + 1) + ' / ' + slides.length;
  const isDark = slides[cur].classList.contains('dark') || slides[cur].classList.contains('terra');
  slideCounter.style.color = isDark ? 'rgba(244,237,227,0.35)' : 'var(--ink-faint)';
}

function go(n) {
  slides[cur].classList.remove('active');
  cur = Math.max(0, Math.min(n, slides.length - 1));
  slides[cur].classList.add('active');
  updateUI();
}

document.addEventListener('keydown', e => {
  if (['ArrowRight','ArrowDown',' '].includes(e.key)) { e.preventDefault(); go(cur + 1); }
  if (['ArrowLeft','ArrowUp'].includes(e.key))         { e.preventDefault(); go(cur - 1); }
  if (e.key === 'Home') go(0);
  if (e.key === 'End')  go(slides.length - 1);
});

let touchStartX = 0;
document.addEventListener('touchstart', e => { touchStartX = e.touches[0].clientX; }, { passive: true });
document.addEventListener('touchend', e => {
  const dx = e.changedTouches[0].clientX - touchStartX;
  if (Math.abs(dx) > 50) go(dx < 0 ? cur + 1 : cur - 1);
});

updateUI();
</script>
</body>
</html>
```

---

## 禁止事项

- 不用 Arial、Inter、Roboto、system-ui
- 不在标题下加下划线装饰（AI 生成感最强的特征）
- 不用图片做装饰背景
- 不把每张幻灯片塞满——留白是设计的一部分
- 不补充原文没有的数据、解释或判断

---

## 完成后自查

1. 有没有哪句话是原文没有的？→ 删掉
2. 有没有哪张幻灯片同时说了两件事？→ 拆开
3. 有没有哪句话换个行业也能用？→ 删掉
4. 图片是否全部 base64 嵌入？
5. 键盘翻页和导航点是否正常工作？
6. 输出文件路径是否正确（与源文件同目录，扩展名 .html）？

---

---

## PPTX 分支（仅在 `--pptx` 时执行，以下替代第一步至第三步）

### 第一步（PPTX）：读取源文件

与 HTML 分支相同：`.docx` 解包提取文本，`.md` 直接读取。图片保存到 `/tmp/` 供后续嵌入。

### 第二步（PPTX）：规划幻灯片

与 HTML 分支的内容原则和幻灯片类型**完全相同**（dark / terra / cream 三种类型，节奏规则不变）。

### 第三步（PPTX）：生成 Python 脚本并执行

#### 3.1 依赖检查

```bash
python3 -c "import pptx" 2>/dev/null || pip3 install python-pptx
```

#### 3.2 生成脚本

根据规划好的幻灯片内容，生成一个完整 Python 脚本，写入 `/tmp/gen_pptx_<timestamp>.py`，再执行：

```bash
python3 /tmp/gen_pptx_<timestamp>.py
```

脚本模板如下（填入实际幻灯片内容）：

```python
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN
from pptx.oxml.ns import qn
from lxml import etree
import copy, os

# ── 颜色常量（与 HTML CSS 变量一一对应）──────────────────────────────────
CREAM     = RGBColor(0xF4, 0xED, 0xE3)
TERRA     = RGBColor(0xB0, 0x5A, 0x42)
TERRA_DIM = RGBColor(0xC4, 0x79, 0x5F)
INK       = RGBColor(0x23, 0x15, 0x10)
INK_SOFT  = RGBColor(0x5C, 0x3D, 0x30)
INK_FAINT = RGBColor(0x9E, 0x7B, 0x6E)
RULE      = RGBColor(0xD4, 0xBF, 0xB0)

W = Inches(13.33)
H = Inches(7.5)

prs = Presentation()
prs.slide_width  = W
prs.slide_height = H

blank_layout = prs.slide_layouts[6]  # 完全空白布局

# ── 工具函数 ─────────────────────────────────────────────────────────────

def set_bg(slide, color: RGBColor):
    """设置幻灯片背景色"""
    bg = slide.background
    fill = bg.fill
    fill.solid()
    fill.fore_color.rgb = color

def add_textbox(slide, text, left, top, width, height,
                font_name="Georgia", font_size=18, bold=False, italic=False,
                color=INK_SOFT, align=PP_ALIGN.LEFT,
                line_spacing=None, zh_font="Noto Serif SC"):
    """添加文本框，同时设置西文字体和中文字体"""
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    tf.word_wrap = True
    p = tf.paragraphs[0]
    p.alignment = align
    run = p.add_run()
    run.text = text
    f = run.font
    f.name      = font_name
    f.size      = Pt(font_size)
    f.bold      = bold
    f.italic    = italic
    f.color.rgb = color
    # 设置东亚字体（中文）
    rPr = run._r.get_or_add_rPr()
    ea = etree.SubElement(rPr, qn('a:ea'))
    ea.set('typeface', zh_font)
    if line_spacing:
        from pptx.util import Pt as _Pt
        from pptx.oxml.ns import nsmap
        pPr = p._pPr
        if pPr is None:
            pPr = p._p.get_or_add_pPr()
        lnSpc = etree.SubElement(pPr, qn('a:lnSpc'))
        spcPct = etree.SubElement(lnSpc, qn('a:spcPct'))
        spcPct.set('val', str(int(line_spacing * 100000)))
    return txBox

def add_rule(slide, left, top, width=Inches(0.56), height=Pt(1), color=TERRA):
    """添加细分隔线（用扁矩形实现）"""
    shape = slide.shapes.add_shape(
        1,  # MSO_SHAPE_TYPE.RECTANGLE
        left, top, width, height
    )
    shape.fill.solid()
    shape.fill.fore_color.rgb = color
    shape.line.fill.background()  # 无描边
    return shape

def add_alpha_textbox(slide, text, left, top, width, height,
                      font_name="Cormorant Garamond", font_size=120,
                      color_hex="F4EDE3", alpha_pct=20,
                      zh_font="Noto Serif SC"):
    """添加带透明度的文字（用于章节页大数字）"""
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    p = tf.paragraphs[0]
    run = p.add_run()
    run.text = text
    rPr = run._r.get_or_add_rPr()
    # 设置字号
    rPr.set(qn('sz'), str(font_size * 100))
    rPr.set(qn('b'), '0')
    # 设置字体
    latin = etree.SubElement(rPr, qn('a:latin'))
    latin.set('typeface', font_name)
    ea = etree.SubElement(rPr, qn('a:ea'))
    ea.set('typeface', zh_font)
    # 设置带透明度的颜色
    solidFill = etree.SubElement(rPr, qn('a:solidFill'))
    srgbClr   = etree.SubElement(solidFill, qn('a:srgbClr'))
    srgbClr.set('val', color_hex)
    alpha     = etree.SubElement(srgbClr, qn('a:alpha'))
    alpha.set('val', str(alpha_pct * 1000))  # 20% → 20000
    return txBox

# ── 幻灯片生成函数 ────────────────────────────────────────────────────────

def make_dark_slide(prs, title, caption=None):
    """Dark 页：封面或金句页"""
    slide = prs.slides.add_slide(blank_layout)
    set_bg(slide, INK)
    L = Inches(1.0); T_start = Inches(2.5)
    W_box = Inches(10.5)
    if caption:
        add_textbox(slide, caption.upper(), L, T_start, W_box, Inches(0.4),
                    font_size=9, color=RGBColor(0x9E,0x7B,0x6E))
        T_start += Inches(0.55)
    add_rule(slide, L, T_start, color=TERRA_DIM)
    T_start += Inches(0.25)
    add_textbox(slide, title, L, T_start, W_box, Inches(2.5),
                font_name="Cormorant Garamond", font_size=40,
                color=CREAM, line_spacing=1.55)
    return slide

def make_cover_slide(prs, title, subtitle=None, meta=None):
    """封面（dark）：标题 + 副标题 + 日期/来源"""
    slide = prs.slides.add_slide(blank_layout)
    set_bg(slide, INK)
    L = Inches(1.0)
    if subtitle:
        add_textbox(slide, subtitle.upper(), L, Inches(2.2), Inches(10), Inches(0.4),
                    font_size=9, color=RGBColor(0x9E,0x7B,0x6E))
    add_textbox(slide, title, L, Inches(2.8), Inches(9), Inches(2.2),
                font_name="Cormorant Garamond", font_size=44,
                color=CREAM, line_spacing=1.35)
    if meta:
        add_textbox(slide, meta, L, Inches(5.3), Inches(8), Inches(0.6),
                    font_size=9, color=RGBColor(0x9E,0x7B,0x6E))
    return slide

def make_terra_slide(prs, chapter_num, chapter_title):
    """Terra 章节页：大编号 + 章节标题"""
    slide = prs.slides.add_slide(blank_layout)
    set_bg(slide, TERRA)
    L = Inches(1.0)
    add_alpha_textbox(slide, chapter_num, L, Inches(0.6), Inches(6), Inches(3.5),
                      font_size=120, color_hex="F4EDE3", alpha_pct=20)
    add_textbox(slide, chapter_title, L, Inches(4.4), Inches(10), Inches(1.2),
                font_name="Cormorant Garamond", font_size=28,
                color=CREAM, line_spacing=1.4)
    return slide

def make_cream_slide(prs, body_text, caption=None):
    """Cream 内容页：caption → 分隔线 → 正文"""
    slide = prs.slides.add_slide(blank_layout)
    set_bg(slide, CREAM)
    L = Inches(1.0); T = Inches(2.0)
    if caption:
        add_textbox(slide, caption.upper(), L, T, Inches(10), Inches(0.35),
                    font_size=9, color=INK_FAINT)
        T += Inches(0.45)
    add_rule(slide, L, T)
    T += Inches(0.25)
    add_textbox(slide, body_text, L, T, Inches(10.5), Inches(4.0),
                font_size=14, color=INK_SOFT, line_spacing=1.9)
    return slide

def make_image_slide(prs, img_path, caption=None, quote=None):
    """图表页：左文右图"""
    slide = prs.slides.add_slide(blank_layout)
    set_bg(slide, CREAM)
    L = Inches(0.8); T = Inches(1.8)
    if caption:
        add_textbox(slide, caption.upper(), L, T, Inches(5.5), Inches(0.35),
                    font_size=9, color=INK_FAINT)
        T += Inches(0.45)
    add_rule(slide, L, T)
    T += Inches(0.25)
    if quote:
        add_textbox(slide, quote, L, T, Inches(5.5), Inches(3.0),
                    font_name="Cormorant Garamond", font_size=22,
                    color=INK_SOFT, line_spacing=1.55)
    slide.shapes.add_picture(img_path, Inches(7.0), Inches(0.8),
                             Inches(5.8), Inches(5.8))
    return slide

# ══ 在此填入实际幻灯片 ════════════════════════════════════════════════════
# 示例（Claude 应根据内容替换为真实幻灯片）：

make_cover_slide(prs,
    title    = "主标题",
    subtitle = "副标题或来源",
    meta     = "日期 · 数据来源"
)

make_terra_slide(prs, "01", "第一章节名")

make_cream_slide(prs,
    caption   = "Caption 小标题",
    body_text = "正文内容……"
)

make_dark_slide(prs,
    caption = "可选 caption",
    title   = "这里是最重的那一句判断。"
)

# ══ 结束幻灯片定义 ════════════════════════════════════════════════════════

OUTPUT_PATH = "OUTPUT_PATH_PLACEHOLDER"  # Claude 替换为实际输出路径
prs.save(OUTPUT_PATH)
print(f"已保存：{OUTPUT_PATH}")
```

> **注意**：Claude 必须将示例部分替换为根据源文件内容规划的真实幻灯片，删除所有示例占位符。`OUTPUT_PATH_PLACEHOLDER` 替换为实际输出路径（与源文件同目录，扩展名 `.pptx`）。

---

## 禁止事项（PPTX 专项补充）

- 不用 `prs.slide_layouts[非6]` 带入默认占位框——始终用 `blank_layout`
- 不在脚本里 `print` 调试信息（除最终保存确认）
- 不把多段正文塞进同一个 `add_textbox`——超长文本拆成多张幻灯片

---

## 完成后自查（PPTX 专项）

1. 用 Keynote 或 PowerPoint 打开验证排版
2. 中文是否正常显示（无乱码、无方框）
3. 背景色是否正确（dark=`#231510`，terra=`#B05A42`，cream=`#F4EDE3`）
4. 分隔线是否渲染为细线（高度 `Pt(1)`）而非粗条
5. 章节页大数字是否呈半透明效果
6. 输出路径是否正确（与源文件同目录，扩展名 `.pptx`）
