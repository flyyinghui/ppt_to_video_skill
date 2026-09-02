# PPT 元素级动画视频管线（ppt-element-video-pipeline）

把 PPT（`.pptx`）转成带 AI 配音、元素逐个出现动画的 mp4 讲解视频。

**触发条件**：用户要把 PPT/演示文稿转成带配音、元素逐个出现的讲解视频，或提到「PPT 转视频」「元素级动画」「路演视频」「示范应用视频」。

---

## 一、核心架构

```
python-pptx 提取元素 → slides.json
        ↓
DeepSeek 生成解说词 / 读官方 docx → narration.json
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

---

## 二、环境依赖

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

---

## 三、完整步骤

### 1. 提取元素（extract_elements_v3.py）

用 python-pptx 递归遍历所有形状（含 group），导出：

- 图片（`shape.image.blob` 写 PNG）
- 视频（解析 slide rels 找到 media 文件）
- 文本框（坐标、字号、文本）
- autoshape（矩形/圆角矩形/椭圆/箭头/线条/自由形，含填充色/边框/线宽/旋转）
- 原生表格（`sh.has_table`，读 cells）
- 原生图表（`sh.has_chart`，读 plots 的 categories/series/values）

每个文本元素**额外捕获 `font_name`（首 run）、`bold`、`align`（首段）**——这是后续识别「大写序号标题」的必需字段（标题判定 `bold && y≤0.12 && font_pt≥18`）。

### 2. 生成解说词（gen_narration.py / extract_narration_docx.py）

两种模式：

**A. DeepSeek 生成**（用户无逐字稿时）：DeepSeek v4-flash（OpenAI SDK，`model="deepseek-v4-flash"`，`extra_body={"thinking":{"type":"disabled"}}`），每页文字要点 → 50~100 字口语解说词。

**B. 用户提供解说词 docx**（首选，最准）：用 `python-docx` 读 docx，每段对应一页，正则去「（PX）」前缀。每段朗读时长即该页动画时长。

### 3. TTS（gen_tts.py）

edge-tts 中文男声 `zh-CN-YunjianNeural`（正式汇报）或 `zh-CN-XiaoxiaoNeural`（女声），`rate="-5%"` 略慢。每页合成 slideNN.mp3，ffprobe 记录时长。

### 4. 时间轴（gen_timeline.py）

- 无视频页：时长 = 解说音频时长
- 视频页：时长 = min(max(解说, 视频时长), 30)（内插视频 cap 30 秒，用户偏好）
- 记录每页音频文件 + 视频列表（含 play_dur）

### 5. Remotion 渲染（remotion/）

组件：`Root.tsx`（Composition + 总帧数）、`PPTVideo.tsx`（Series 拼接 + 类型）、`Slide.tsx`（元素渲染 + 动画）。

动画与排版规则见 [第五节](#五动画与排版规则用户偏好-2026-09-02)。

### 6. 混音（mix_audio.py）

所有解说段 + 视频原声，ffmpeg `adelay` 到各页起始偏移，视频原声 `atrim` 到 play_dur，最后 `amix=inputs=N:duration=longest:normalize=0` 合成完整音轨。

### 7. 合并

```bash
ffmpeg -y -i video_nosound.mp4 -i final_audio.m4a \
  -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -b:a 192k -shortest 最终.mp4
```

---

## 四、动画与排版规则（用户偏好，2026-09-02 更新）

### 动画规则

- 文字：**不做动画**，静态显示
- 所有元素动画顺序：**从上到下、从左到右**依次淡入（先按 y 行分组 [容差 0.15 英寸]，同行内**文字优先**于图表/表格/图片，再按 x 排序）
- 图表/表格**对应的文字**（标题/标注/图例）：**先于**图表/表格本身出现
- 视频：**同步播放**（多视频同页同时开始，不参与 stagger）
- 重叠元素：按 z 顺序（上下）依次出现
- group 整合元素（图片+文字一体）：作为一组同时显示

### 排版规则

- 顶部大写序号标题（一/二/三/四 + 章节名）= 微软雅黑、**左对齐**、**深蓝色 #1F4E79**、加粗
- 正文长段落 = 微软雅黑、**居中**、**深蓝色 #1F4E79**

`staticFile()` 引用 public/ 下的素材（软链接 assets → public/assets）。
