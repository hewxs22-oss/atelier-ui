# 照片素材管线

需要真实材质（木、蜡、麻、金属、缎、花）的物件，用图像模型生成照片再抠图。SVG 只用来画会变的部分（指针、日期、文字）、需要精确透明度的部分（薄纱），以及形状本身就是图形语言的标志（海星、贝壳）。

## 1. 什么时候用照片、什么时候用 SVG

| 用照片 | 用 SVG / CSS |
|---|---|
| 物件的材质是卖点：钟、火漆、麻绳、剪刀、布、花 | 会随数据变的：时间指针、日期数字、文字 |
| 用户说过「没质感」「塑料感」「假」 | 需要可控透明度的大面积层：薄纱、湿痕 |
| 需要光影、磨损、纤维细节 | 与 logo 同一画法的图形：海星、贝壳 |
| | 纸纹：`feTurbulence` 噪声 + 少量手画元素（雏菊、横线） |

## 2. 生成

- 模型：项目里配置的图像模型（本项目是 302.ai `gpt-image-2`，`/v1/images/generations`）。密钥通过项目的配置函数取（本项目 `config.get_302ai_image_key()`），不直接读密钥文件、不在输出里打印。
- 尺寸：横物件 `1536x1024`，竖物件 `1024x1536`，方物件 `1024x1024`；`quality: high`。
- 多个素材并发生成（线程池 8），失败的单独重跑；已生成的跳过。
- 背景：
  - **白底**：边缘清楚、内部没有白色的物件（金属剪刀）。
  - **绿幕**（`#00b140`）：浅色、白色、半透明或细碎的物件（象牙缎、白手套、薰衣草细茎）。

提示词模板（按需替换花括号）：

```
Overhead studio product photograph of a single {物件描述}, lying flat, {摆放方向，例如 perfectly horizontal, handles left, tips right},
centered, filling about 80% of the frame. {材质细节：real metal texture with fine engraving and patina / soft satin sheen / ...}.
Background: a flat solid uniform bright green chroma-key colour (#00b140), nothing else, no shadow, no text.
```

要点：

- 写清方向（「刀尖朝右」「花在左、梗在右」），后面拼进页面时少转一次。
- 写清要什么不要什么（「not crocheted, not plastic」「no lace」「empty pinch」）。
- 一次出 2 个变体放一起比，再选。

## 3. 抠图

绿幕（适合浅色物件）：

```python
a = np.asarray(img.convert('RGB')).astype(float)
r, g, b = a[..., 0], a[..., 1], a[..., 2]
green = g - np.maximum(r, b)                  # 绿幕处约 +120，物件处 ≤ 0
alpha = np.clip((40 - green) / 40, 0, 1)      # 0..40 是过渡带；细茎用 34
mask = Image.fromarray((alpha * 255).astype('uint8')).filter(ImageFilter.MinFilter(3)).filter(ImageFilter.GaussianBlur(.6))
spill = np.clip(g - (r + b) / 2, 0, None); a[..., 1] -= spill * .9    # 去绿边
```

白底（适合深色、金属物件）：近白且近灰的像素做漫水填充，只去掉和画面边缘连通、或面积大于 250 像素的白块（保住把手环里的小孔），再按 alpha 从白底反解颜色去白边：`rgb = (rgb - 255 * (1 - α)) / α`。

细茎（薰衣草）不要做 `MinFilter`，会把茎吃掉。

## 4. 局部重上色

不重新生成，用掩码改一部分颜色：

- 先按色相选出要改的像素，例如「偏黄绿的梗」：`b < .9 * g` 且 `r > .55 * g`，紫色的花 `b` 高，不会被选中。
- 按原亮度映射到目标色，保住明暗：`target * (0.45 + 0.75 * lum)`。
- 整体轻调色（象牙缎 → 极淡粉）：绿、蓝通道各乘 `.90 / .95`。用户说「看着廉价」时，调色救不回来，要重新生成材质（例：贝母光泽真丝缎）。

## 5. 导出与引用

- 格式：透明 WebP，`quality 86–88, method 6`。
- 尺寸：按显示尺寸的 2 倍左右。卡片角上的小剪刀 200px 宽，弹窗大图 720px，布 1100px。
- **小图**（几 KB）：内嵌 data URI。
- **大图**：放静态目录走路由（本项目 `tools/review_assets/` → `/assets/ui/`），地址加 `?v=N`，换图就 +1，否则浏览器拿旧缓存。
- 同一物件在动画里会被放大时，从一开始就用大图，不要先放小图再替换（否则飞行时是糊的）。
- 原图和中间产物放 scratchpad，不进仓库。

## 6. 和页面拼合

- 投影只给物件本身：`drop-shadow` 加在图片上，不加在装它的容器上（见 `performance.md`）。
- 照片和画风不搭时，先试：
  - 统一暖色：`sepia(.2) saturate(.88)`。
  - 加一圈 1–2px 奶白裁切边（贴纸感）。
  - 加贴着底面的短投影（「真的放在布上」）。
- 状态色（雨天）用整体滤镜改，不重新生成：`sepia(1) saturate(1.4) hue-rotate(-12deg) brightness(.58)`，保留纹理、不要死黑。
- 物件要盖住画布上的东西（麻绳），在画布上按图片的 alpha 用 `destination-out` 擦掉；图片晚到时，`onload` 后再擦一次。
