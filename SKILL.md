---
name: ppt-element-video-pipeline
description: 把 PPT 转成「元素级动画视频」的完整管线。将每页的图片、表格、图表、内插视频提取为独立元素，用 Remotion 做逐个出现动画，DeepSeek 生成解说词 + edge-tts 配音，按音频时长对齐动画，最终渲染 mp4。触发条件：用户要把 PPT/演示文稿转成带配音、元素逐个出现的讲解视频，或要「PPT 转视频」「元素级动画」「路演视频」。
---

# PPT 元素级动画视频管线

把 PPT（.pptx）转成带 AI 配音、元素逐个出现动画的 mp4 讲解视频。

## 核心架构

```
python-pptx 提取元素 → slides.json
        ↓
DeepSeek 生成解说词 → narration.json
        ↓
edge-tts 合成配音 → audio/slideNN.mp3 + durations.json
        ↓
视频时长对齐（含内插视频）→ final_timeline.json
        ↓
Remotion 渲染画面（元素级动画）→ video_nosound.mp4
        ↓
ffmpeg 混音（解说 + 视频原声）→ final_audio.m4a
        ↓
ffmpeg mux → 最终 mp4
```

## 依赖

```bash
# Python（Hermes venv）
pip install python-pptx edge-tts openai

# Node/Remotion（用 bun 装的 npm，PATH 加 /root/.bun/bin）
npm install @remotion/cli@^4 @remotion/player@^4 react@18 react-dom@18 --registry=https://registry.npmmirror.com
npm install-scripts approve esbuild && npm rebuild esbuild  # esbuild postinstall 被 npm 阻止必须手动批准

# Chrome Headless Shell（Remotion 渲染必需）
npx remotion browser ensure

# 系统
apt-get install -y ffmpeg
```

## 步骤

### 1. 提取元素（extract_elements_v2.py）

用 python-pptx 递归遍历所有形状（含 group），导出：
- 图片（`shape.image.blob` 写 PNG）
- 视频（解析 slide rels 找到 media 文件）
- 文本框（坐标、字号、文本）
- autoshape（矩形/圆角矩形/椭圆/箭头/线条/自由形，含填充色/边框/线宽/旋转）
- 原生表格（`sh.has_table`，读 cells）
- 原生图表（`sh.has_chart`，读 plots 的 categories/series/values）

### 2. 生成解说词（gen_narration.py / extract_narration_docx.py）

两种模式：

**A. DeepSeek 生成**（用户无逐字稿时）：DeepSeek v4-flash（OpenAI SDK，`model="deepseek-v4-flash"`，`extra_body={"thinking":{"type":"disabled"}}`），每页文字要点 → 50~100 字口语解说词。

**B. 用户提供解说词 docx**（首选，最准）：用 `python-docx` 读 docx，每段对应一页，正则 `^（P\d+）\s*(.*)$` 提取页码+正文（去掉「（PX）」标记不念）。43 段 → narration.json。每段朗读时长即该页动画时长。

### 3. TTS（gen_tts.py）

edge-tts 中文男声 `zh-CN-YunjianNeural`（正式汇报）或 `zh-CN-XiaoxiaoNeural`（女声），`rate="-5%"` 略慢。每页合成 slideNN.mp3，ffprobe 记录时长。

### 4. 时间轴（gen_timeline.py）

- 无视频页：时长 = 解说音频时长
- 视频页：时长 = min(max(解说, 视频时长), 30)（内插视频 cap 30 秒，用户偏好）
- 记录每页音频文件 + 视频列表（含 play_dur）

### 5. Remotion 渲染（remotion/）

组件：`Root.tsx`（Composition + 总帧数）、`PPTVideo.tsx`（Series 拼接 + 类型）、`Slide.tsx`（元素渲染 + 动画）。

动画规则（用户偏好，2026-09-02 更新）：
- 文字：**不做动画**，静态显示
- 所有元素动画顺序：**从上到下、从左到右**依次淡入（先按 y 行分组 [容差 0.15 英寸]，同行内**文字优先**于图表/表格/图片，再按 x 排序）
- 图表/表格**对应的文字**（标题/标注/图例）：**先于**图表/表格本身出现
- 视频：**同步播放**（多视频同页同时开始，不参与 stagger）
- 重叠元素：按 z 顺序（上下）依次出现
- group 整合元素（图片+文字一体）：作为一组同时显示

排版规则（用户偏好，2026-09-02 确立）：
- 顶部大写序号标题（一/二/三/四 + 章节名）= 微软雅黑、**左对齐**、**深蓝色 #1F4E79**、加粗
- 正文长段落 = 微软雅黑、**居中**、**深蓝色 #1F4E79**

`staticFile()` 引用 public/ 下的素材（软链接 assets → public/assets）。

### 6. 混音（mix_audio.py）

43 段解说 + 6 段视频原声，ffmpeg `adelay` 到各页起始偏移，视频原声 `atrim` 到 play_dur，最后 `amix=inputs=N:duration=longest:normalize=0` 合成完整音轨。

### 7. 合并

```bash
ffmpeg -y -i video_nosound.mp4 -i final_audio.m4a -c:v copy -c:a aac -b:a 192k -shortest 最终.mp4
```

## 关键陷阱（本会话踩坑实录）

1. **group 坐标是绝对的**：python-pptx 对 group 内子 shape 的 `left/top` 返回**绝对坐标**（已含 group 偏移），不要累加，否则元素越界（如 14.75 英寸 > 10 英寸页面）。
2. **autoshape 类型用 `.name`**：`str(ast)` 返回 `"RECTANGLE (1)"` 带枚举序号，必须用 `getattr(ast, 'name', ...)` 或 `str(ast).split(' ')[0]` 才能匹配。
3. **LINE(9)/FREEFORM(5) 是独立 shape_type**：不是 AUTO_SHAPE(1)，`auto_shape_type` 返回 None，需单独分支处理。
4. **原生 table/chart**：`has_table`/`has_chart` 判断，chart 数据从 `ch.plot.categories` + `ch.plot.series[].values` 读；chart 可能在 group 内需递归。
5. **esbuild 必须批准 postinstall**：npm 默认阻止 esbuild 的 `node install.js`，导致 Remotion 渲染失败。`npm install-scripts approve esbuild && npm rebuild esbuild`。
6. **Chrome 缺失**：Remotion 渲染需要 Chrome Headless Shell，`npx remotion browser ensure` 下载（91.9MB）。WSL 无 GUI，用 headless shell。
7. **中文字体**：WSL 默认只有文泉驿微米黑/Droid Sans Fallback。两种 Windows 字体按需复制 + `fc-cache -f`：**等线**用 `Deng*.ttf`（单 ttf），**微软雅黑**用 `msyh.ttc`+`msyhbd.ttc`（ttc 集合字体，Regular+Bold）。复制后必须**重新渲染**才生效（Chrome 会缓存字体）。微软雅黑字体栈：`'Microsoft YaHei', '微软雅黑', 'WenQuanYi Micro Hei', sans-serif`。详见 `references/title-body-styling-and-msyh-font.md`。
8. **微信发视频限流**：send_message 工具连续发媒体会触发 rate limit（ret=-2）。完整版（>100MB）直接存 D 盘让用户打开，微信只发压缩预览版（640 宽 crf 30，~12MB）。
9. **PPT 无备注时解说词来源**：用 DeepSeek 把每页文字要点扩写成口语解说词（备注几乎全空时）；或直接朗读原文；或用户提供逐字稿。
10. **视频页音频重叠**：解说词（引导语）与视频原声混音，视频原声需 `atrim` 到页时长，否则泄漏到下一页。
11. **Remotion 渲染的视频自带静音音频流**：当组件里用 `OffthreadVideo` 播放内插视频时，Remotion 渲染输出的 mp4 会自带一个静音 aac 音频流。合并时若不加 `-map`，ffmpeg 默认取第一个输入的音频流（即这个静音流），导致成品静音。**必须显式指定**：`ffmpeg -i video.mp4 -i audio.m4a -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -shortest out.mp4`。诊断信号：成品 mp4 的 volumedetect 显示 -91dB（静音），而单独的 audio 文件 -23dB 正常。
12. **越界元素用「平移优先」而非「等比缩放」**：等比缩放会把右侧/底部越界的图表缩得几乎看不见（如 P13 柱状图 5.39→2.34 英寸、P29 图 3.18→0.05 英寸），还会产生负尺寸 bug（x>10 时 scale 为负）。正确做法：负坐标→平移到 0；`x+w>10` 时**左移** `x = 10-w`（保持尺寸，仅当 w>10 才缩小）；`y+h>7.5` 时**上移** `y = 7.5-h`；最后坐标 round 到 0.02 英寸网格对齐统一排版。
13. **图表/表格不缩小、对齐统一排版**：用户要求图表表格保持合理大小且对齐统一，不因越界被缩得参差不齐。「平移优先 + 网格对齐」是标准解法（见 #12）。
12. **越界元素用「平移优先」，绝不用「等比缩放」**：等比缩放会把右侧/底部越界的图表缩得几乎看不见（如 P13 柱状图 5.39→2.34 英寸、P29 图 3.18→0.05 英寸），还会因 x 越界产生负尺寸 bug（w 变负）。正确做法：负坐标平移到 0；`x+w>PAGE_W` 时左移到 `x=PAGE_W-w`（保持尺寸），仅当元素本身比页面宽才缩小；底部越界同理上移。最后坐标 round 到 0.02 英寸网格做统一对齐排版。
13. **解说词优先用官方 docx**：用户常提供「项目示范应用解说词YYYYMMDD.docx」，每段对应一页 PPT，段首有「（PX）」标记（不念）。提取用正则 `^（P(\d+)）\s*(.*)$` 去前缀，按页号建 narration.json，再走 edge-tts 录音。每页动画时长 = 该段朗读时长。没有官方 docx 时才用 DeepSeek 从页面文字扩写（见陷阱 9）。
11. **越界规范化 / group 整合分组 / 逐页帧号计算 / 等线字体复制**：实现级细节见 `references/implementation-details.md` —— ①元素越界（本案例 60 个）等比缩放 + 负坐标平移；②「图片+文字一体」靠 group 的 z 作 gid 归属识别；③长视频（118s 录屏）原声必须 atrim 否则盖过后页；④每页时长不同，验证单页帧号 = 累计时长 × fps（非固定 180 帧/页）；⑤等线字体从 `/mnt/c/Windows/Fonts/Deng*.ttf` 复制到 WSL + fc-cache，复制后须重新渲染才生效。

## 复用脚本位置

脚本在本会话 `/tmp/ppt_pipeline/`，但 /tmp 重启即清空，应随项目持久化。核心脚本：`extract_elements_v2.py`（元素提取）、`gen_narration.py`（解说词）、`gen_tts.py`（配音）、`gen_timeline.py`（时间轴）、`mix_audio.py`（混音），Remotion 组件 `remotion/src/{Root,PPTVideo,Slide}.tsx`。平移优先对齐排版 + docx 解说词提取的完整可复用代码见 `references/layout-alignment-and-narration-docx.md`。

**标题/正文排版 + 微软雅黑字体 + 像素验证**：用户常要求「大写序号标题（一/二/三/四 + 章节名）= 微软雅黑、左对齐、深蓝 #1F4E79；正文长段落 = 居中、微软雅黑」。识别规则（bold + y≤0.12 + font_pt≥18）、normalize 后打 `role` 标签、Slide.tsx 两个 case（text/rect）都要按 role 分支、PIL 像素采样验证颜色与左对齐（vision 子代理会超时），全见 `references/title-body-styling-and-msyh-font.md`。
