# 核对：每一轮都要看到、量到

只说「改好了」不算。每一轮至少：

1. 改动处特写（2 倍像素密度）。
2. 整页。
3. 两种窗口尺寸（如 1500×900 和 1280×720）。
4. 每个状态（雨天、简约、收起、弹窗开着）。
5. 有动效：取中间帧和结束帧；确认结束后动画列表里没有它。
6. 能量化的就量：尺寸、是否溢出、是否被选中、主线程占用。

## Playwright + 路由替身

不碰真服务和登录，用真实数据（先存一份接口返回到临时目录）喂页面。本项目用独立的 Playwright 解释器；拼图用系统 Python 的 PIL。

```python
from playwright.sync_api import sync_playwright
html = open('tools/review_gloss.html').read(); arts = open(T + '/articles.json').read()
def route(r):
    u = r.request.url
    if '/api/articles' in u: return r.fulfill(body=arts, content_type='application/json')
    if '/api/' in u: return r.fulfill(body='{}', content_type='application/json')
    if '/assets/ui/' in u:
        return r.fulfill(body=open('tools/review_assets/' + u.split('/assets/ui/')[1].split('?')[0], 'rb').read(), content_type='image/webp')
    r.fulfill(body=html, content_type='text/html')
with sync_playwright() as p:
    b = p.chromium.launch(); pg = b.new_page(viewport={'width': 1500, 'height': 900}, device_scale_factor=2)
    pg.route('http://x.test/**', route); pg.goto('http://x.test/')
    pg.evaluate("localStorage.setItem('xb_rain','0'); showHome()"); pg.wait_for_timeout(2500)
    pg.screenshot(path=S + 'shot.png', clip={'x': 0, 'y': 0, 'width': 1500, 'height': 220}, timeout=15000)
```

- 截图单独设 `timeout=15000`，整个脚本用 `timeout 120` 包住，不要 `sleep` 等待。
- 收集 `pageerror` 和 console error；改完先用 `new Function` 逐段解析内联脚本，查语法错。

## 量化检查

- 一栏放几张、高度是否统一：

```js
[...document.querySelectorAll('.home-col-list')].map(l => {
  const lr = l.getBoundingClientRect();
  const cs = [...l.querySelectorAll('.home-card')].map(c => c.getBoundingClientRect());
  return { h: [...new Set(cs.map(c => Math.round(c.height)))], visible: cs.filter(c => c.bottom <= lr.bottom + 1).length };
})
```

- 文字是否被裁：`el.scrollWidth > el.clientWidth + 1`。
- 双击后有没有选中：`getSelection().toString() === ''`。
- 动效是否一次就停：触发后 `document.getAnimations().map(a => a.animationName)`，等动画时长后再取一次。
- 素材晚到时层级是否还对：在路由里给这张图加 1.5 秒延迟再截图。

## 抖动检测

headless 下合成行为和真机不同，复现「隔几秒动一下」要用有界面的浏览器：

```python
b = p.chromium.launch(headless=False, args=['--window-position=2000,2000'])
pg = b.new_page(viewport={'width': 1500, 'height': 900}, device_scale_factor=2)
# 打开弹窗后，每 250ms 截同一区域 40 次，比较字节和元素 rect
```

位置（rect）不变、像素差只有 ±2 的是 GPU 噪点，不是抖动。没复现时如实告诉用户，请对方录屏，并说明浏览器、是否外接屏、抖动是否和别的动画同步。

## 主线程占用

```python
c = pg.context.new_cdp_session(pg); c.send('Performance.enable')
m0 = {x['name']: x['value'] for x in c.send('Performance.getMetrics')['metrics']}
pg.wait_for_timeout(5000)
m1 = {x['name']: x['value'] for x in c.send('Performance.getMetrics')['metrics']}
busy = (m1['TaskDuration'] - m0['TaskDuration']) / 5   # 应接近 0
```

## 给用户选的对比图

多个候选用同一个截图区域，横向拼成一张，上方标 A / B / C，发给用户。用户选定后再同步风格文档。
