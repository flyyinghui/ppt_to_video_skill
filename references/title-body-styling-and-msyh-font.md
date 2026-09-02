# 标题/正文角色分类 + 微软雅黑字体 + 渲染帧像素验证（v3 会话验证，2026-09-02）

官方汇报 PPT 转视频时的排版要求（用户偏好，与「等线」不同，本次用「微软雅黑」）：

## 1. 微软雅黑（Microsoft YaHei）字体，非等线

字体文件在 Windows（`.ttc` 集合字体，不是单 `.ttf`）：

```bash
mkdir -p /usr/share/fonts/truetype/msyh
cp /mnt/c/Windows/Fonts/msyh.ttc /mnt/c/Windows/Fonts/msyhbd.ttc /usr/share/fonts/truetype/msyh/
fc-cache -f
fc-list | grep -i "yahei\|微软雅黑"   # 验证：出现 Microsoft YaHei / 微软雅黑 Regular + Bold
```

- `msyh.ttc` = 微软雅黑 Regular，`msyhbd.ttc` = Bold，`msyhl.ttc` = Light。
- 等线用 `Deng*.ttf`（单 ttf），微软雅黑用 `msyh*.ttc`（ttc 集合），路径和文件类型都不同，别混。
- Remotion 字体栈（微软雅黑放最前）：

```ts
const FONT = "'Microsoft YaHei', '微软雅黑', 'WenQuanYi Micro Hei', 'Droid Sans Fallback', sans-serif";
```

## 2. 标题「大写序号标题」识别 + 深蓝左对齐

用户说的「位于最顶层的大写序号标题」= 每页顶部的章节栏，由**两个独立 rect** 组成：
- 章节序号 rect：文本 = 「一/二/三/四」单字，font_pt≈27，bold
- 章节名 rect：文本 = 「项目背景」「京津冀城市群集成示范」等，font_pt≈21-24，bold
- 两者都在 y≈0.0-0.06 英寸（最顶），`fill=None`（透明，无背景色块）

识别规则（在提取后的 normalize/classify 步骤打 `role` 标签）：

```python
def classify(item):
    txt = (item.get("text") or "").strip()
    if not txt: return None
    y = item.get("y", 0); fp = item.get("font_pt") or 0; bold = item.get("bold", False)
    if bold and y <= 0.12 and fp >= 18:
        return "title"     # 大写序号标题
    return "body"          # 其余文本一律居中
```

渲染样式（Slide.tsx 里按 `item.role` 分支）：
- **title**：微软雅黑 + `fontWeight: bold` + `textAlign: left` / `justifyContent: flex-start` + 深蓝 `#1F4E79`
- **body**（含正文长段落）：微软雅黑 + `textAlign: center` / `justifyContent: center` + 深蓝 `#1F4E79`

```tsx
const TITLE_COLOR = '#1F4E79';  // 深蓝（标题 + 正文长段落统一深蓝，2026-09-02 更新）
const isTitle = item.role === 'title';
// text kind:
justifyContent: isTitle ? 'flex-start' : 'center',
textAlign: isTitle ? 'left' : 'center',
color: TITLE_COLOR,              // 标题和正文都深蓝
fontWeight: isTitle ? 'bold' : 'normal',
```

注意：标题和正文都可能落在 `rect`（含文字的 autoshape）或 `text`（纯文本框）两种 kind 里，**两个 case 都要加 role 分支**，不能只改 text case。

TypeScript 侧 `SlideItem` 类型要加字段：`role?: 'title' | 'body' | null; font_name?: string | null; bold?: boolean; align?: string | null;`

## 3. 渲染帧像素采样验证（替代 vision 子代理）

vision 子代理对 5 张 PNG 逐张 `vision_analyze` 会超时（600s / 23 次 API 调用后 timeout）。改用 PIL 像素采样快速自检：

```python
from PIL import Image
im = Image.open('test_p13.png').convert('RGB')
w, h = im.size
region = im.crop((0, 0, w, 40))   # 标题区 y=0~40px（0~0.2 英寸 × SCALE 192）
px = list(region.getdata())
# 深蓝像素占比（>0 即标题颜色生效）
dark_blue = sum(1 for r,g,b in px if abs(r-0x1F)<40 and abs(g-0x4E)<40 and abs(b-0x79)<50)
# 左对齐校验：深蓝像素 x 分布，均值 < w/2 即左对齐
xs = [x for x in range(w) for y in range(40)
      if (lambda p: abs(p[0]-0x1F)<40 and abs(p[1]-0x4E)<40 and abs(p[2]-0x79)<50)(px[y*w+x])]
mean_x = sum(xs)/len(xs)   # < 960（半宽）说明左对齐
```

判断标准：标题区深蓝像素占比 1-6%（越长的章节名占比越高），x 均值 < 画布半宽 = 左对齐成立；封面/尾页（无大写序号标题）标题区深蓝像素 = 0 属正常。

## 4. docx 解说词提取（全角/半角括号都支持）

用户 docx 段首「（P1）」是全角括号，但别写死全角，用兼容正则：

```python
m = re.match(r'^[（(]P(\d+)[）)]\s*(.*)$', t)   # 同时匹配（P1）和 (P1)
```

版本迭代：-2 → -3 时页数从 43 → 46（解说词从 42 → 46 段），幻灯片数 == 解说词段数，逐页一一对应。

## 5. 视频页混音防静音（-map 显式选流）

Remotion 用 `OffthreadVideo` 渲染的内插视频会让输出 mp4 **自带一个静音 aac 流**。合并时必须显式 `-map` 选画面流 + 解说音轨流，否则 ffmpeg 默认取第一个输入的音频流（静音流）→ 成品无声。

```bash
ffmpeg -y -i video_nosound.mp4 -i final_audio.m4a \
  -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -b:a 192k -shortest 最终.mp4
```

验证非静音：`ffmpeg -i 最终.mp4 -map 0:a:0 -af volumedetect -f null -` 看 `mean_volume` ≈ -23dB（正常），若 -91dB 即静音（说明 -map 没生效）。

## 6. 动画顺序：从上到下、从左到右 + 文字先于图表/表格（2026-09-02 更新）

旧的排序只按 `z` 顺序（PPT 叠放层级），用户要求改为视觉顺序「从上到下、从左到右」，且图表/表格对应的文字（标题/标注/图例）先于图表/表格本身出现。

Slide.tsx 排序逻辑（替换原来的 `sort((a,b) => a.z - b.z)`）：

```tsx
// 文字类元素：纯文本框，或带文字的 autoshape（rect/roundrect/oval 等）
const isTextItem = (it: SlideItem) =>
  it.kind === 'text' ||
  (['rect', 'roundrect', 'oval', 'diamond', 'triangle', 'freeform'].includes(it.kind) && !!it.text);

const sorted = [...others].sort((a, b) => {
  const aIsText = isTextItem(a);
  const bIsText = isTextItem(b);
  // 1. 文字先于图表/表格/图片（无论 y 位置）
  if (aIsText !== bIsText) return aIsText ? -1 : 1;
  // 2. 同类内：先按 y 从上到下（行容差 0.15 英寸 ≈ 字号高度）
  if (Math.abs(a.y - b.y) > 0.15) return a.y - b.y;
  // 3. 同行内：按 x 从左到右
  return a.x - b.x;
});
```

要点：
- 行容差 `0.15` 英寸：y 差 ≤ 0.15 视为「同一行」，只按 x 排序；> 0.15 视为不同行，按 y 排序。
- 「文字先于图表/表格」用 `isTextItem` 二元判断实现：只要元素含文字（text kind 或带 text 的 autoshape），就排在所有图片/表格/图表之前。这保证图表的标题/标注在图出现前已显示。
- 标题（大写序号）本身是文字元素且 y 最小，天然排最前，无需特殊处理。
- 视频仍不参与 stagger（同步播放），背景大图（面积 > 画布 38%）单独最先淡入。
