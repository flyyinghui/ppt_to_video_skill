# 实现级细节（2026-09-01 会话验证）

## 1. 越界规范化（normalize）

PPT 元素常超出画布（本案例 60 个元素 x+w>10 或 y+h>7.5）。提取后统一 normalize：

```python
PAGE_W, PAGE_H = 10.0, 7.5  # 4:3 画布英寸

def normalize(item):
    # 负坐标平移裁剪（左/上越界）
    if item["x"] < 0:
        item["w"] += item["x"]; item["x"] = 0
    if item["y"] < 0:
        item["h"] += item["y"]; item["y"] = 0
    # 等比缩放到边界内（保持左上角，不裁切内容）
    sx = (PAGE_W - item["x"]) / item["w"] if item["w"] > 0 else 1
    sy = (PAGE_H - item["y"]) / item["h"] if item["h"] > 0 else 1
    scale = min(1.0, sx, sy)
    if scale < 1.0:
        item["w"] *= scale; item["h"] *= scale
```

注意：等比缩放会整体缩小元素（如 P13 柱状图 w=5.39→2.34 英寸）。用户明确接受"放大或缩小"来保证不越界。

## 2. group_id 整合分组（识别"图片+文字一体"）

用户要求"多个图片文字整合在一起的就按一张图一起显示"。识别靠 group 归属：

```python
zcount = [0]
def walk(shapes, out, slide_dir, gid=None):
    for sh in shapes:
        zcount[0] += 1
        z = zcount[0]
        if st == MSO_SHAPE_TYPE.GROUP:
            walk(sh.shapes, out, slide_dir, gid=z)  # group 的 z 作为 gid
            continue
        # ... 每个 append 都带 "group": gid
```

Remotion 侧按 group 分组：`group=None` 的元素各自独立，同 `group` 的元素合并一组，组内同帧淡入（不 stagger）。组排序 = 组内最小 x（从左到右）。

## 3. 视频原声 atrim 防泄漏

视频页时长 cap 30s 后，长视频（如 118s 录屏）的原声必须截断，否则泄漏到后续页：

```python
# mix_audio.py 里，视频原声
f"[{idx}:a]atrim=duration={play_dur},asetpts=PTS-STARTPTS,adelay={start_ms}|{start_ms}[{label}]"
# play_dur = min(视频时长, 30)，从 final_timeline.json 的 play_dur 字段读
```

不 atrim 的后果：amix duration=longest 会让音轨长度 = 最长视频原声，118s 录屏声音会盖过后面的解说。

## 4. 逐页帧号计算（验证单页时）

每页时长不同（按音频对齐），验证/渲染单页时帧号不能用固定值：

```python
fps = 30; cur = 0.0
for t in timeline:
    if t['page'] == target_page:
        mid_frame = int((cur + t['duration'] * 0.6) * fps)
    cur += t['duration']
```

`npx remotion still PptRoadshow out.png --frame=<mid_frame>` 渲染该页 60% 处（大部分元素已出现）。

## 5. 等线字体复制到 WSL

用户要求"等线"字体（Windows 字体，WSL 默认无）：

```bash
mkdir -p /usr/share/fonts/truetype/dengxian
cp /mnt/c/Windows/Fonts/Deng*.ttf /usr/share/fonts/truetype/dengxian/  # Deng/Dengb/Dengl
fc-cache -f
fc-list | grep -i dengxian  # 验证
```

Remotion 字体栈：`"'DengXian', '等线', 'WenQuanYi Micro Hei', 'Droid Sans Fallback', sans-serif"`。注意：复制字体后要**重新渲染**才生效（Chrome 进程会缓存字体）。

## 6. 视频 cap 30 秒（用户偏好）

内插视频（录屏/演示）最长播 30 秒就切页：

```python
VIDEO_MAX = 30.0
duration = min(max(audio_dur, vid_max), VIDEO_MAX)  # 视频页
```

这是用户 2026-09-01 明确要求："视频播放不超过30秒就切换"。
