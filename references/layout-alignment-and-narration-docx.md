# 平移优先对齐排版 + docx 解说词提取（可复用代码）

## 1. 越界元素「平移优先」规范化（替代等比缩放）

等比缩放的坑：右侧/底部越界的元素会被缩得几乎看不见（P13 柱状图 5.39→2.34 英寸），
且 `x+w > PAGE_W` 时 `(PAGE_W-x)/w` 为负，w 变负产生负尺寸 bug。

正确做法——平移保持原始尺寸：

```python
PAGE_W, PAGE_H = 10.0, 7.5  # 4:3 画布（英寸）

def normalize(item):
    # 修复负尺寸（x 越界导致 w 变负）
    if item["w"] < 0:
        item["x"] += item["w"]; item["w"] = -item["w"]
    if item["h"] < 0:
        item["y"] += item["h"]; item["h"] = -item["h"]
    # 负坐标平移
    if item["x"] < 0: item["x"] = 0
    if item["y"] < 0: item["y"] = 0
    # 右侧越界：左移保持尺寸，仅当比页面宽才缩小
    if item["x"] + item["w"] > PAGE_W:
        if item["w"] <= PAGE_W:
            item["x"] = PAGE_W - item["w"]
        else:
            item["w"] = PAGE_W; item["x"] = 0
    # 底部越界：上移保持尺寸
    if item["y"] + item["h"] > PAGE_H:
        if item["h"] <= PAGE_H:
            item["y"] = PAGE_H - item["h"]
        else:
            item["h"] = PAGE_H; item["y"] = 0
    # 网格对齐统一排版（round 到 0.02 英寸）
    item["x"] = round(max(item["x"], 0.0), 3)
    item["y"] = round(max(item["y"], 0.0), 3)
    item["w"] = round(max(item["w"], 0.02), 3)
    item["h"] = round(max(item["h"], 0.02), 3)
```

## 2. 官方解说词 docx 提取（每段对应一页，去「（PX）」前缀）

```python
import re, json
from docx import Document

doc = Document("项目示范应用解说词20260831.docx")
narration = {}
for p in doc.paragraphs:
    t = p.text.strip()
    if not t:
        continue
    m = re.match(r'^（P(\d+)）\s*(.*)$', t)   # 段首「（P1）」「（P2）」...
    if m:
        narration[str(int(m.group(1)))] = m.group(2).strip()

with open("narration.json", "w", encoding="utf-8") as f:
    json.dump(narration, f, ensure_ascii=False, indent=1)
```

之后直接走 edge-tts（`zh-CN-YunjianNeural` 男声，`rate="-5%"`）每段合成 slideNN.mp3，
每页动画时长 = 该段朗读时长（ffprobe 读时长）。视频页仍取 `min(max(朗读, 视频时长), 30)` cap 30 秒。

## 3. 每页时长对齐

```python
# 无视频页：时长 = 朗读时长；视频页：min(max(朗读, 视频), 30)
VIDEO_MAX = 30.0
duration = min(max(audio_dur, vid_max), VIDEO_MAX) if has_video else audio_dur
```
