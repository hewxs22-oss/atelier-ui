# atelier-ui

一个 Claude Code / Agent Skill：做**有质感、独一份**的前端界面，并按真人反馈一轮轮收敛。

A Claude Code / agent skill for designing tactile, one-of-a-kind front-end UIs and iterating them against real human feedback.

## 它是什么

从一个内部审稿页主页两天、上百轮的改版里提炼出来。分两层：

- **通用层**（`SKILL.md` + `references/`）：任何风格都用得上的方法。
  - 开工前先定骨架和用户路线，写一份风格 prompt 文档。
  - 每轮迭代：定位 → 解读反馈 → 做 → 截图核对 → 说清楚 → 同步文档 → 快速回退。
  - 十条通用原则：内容是主角、整体感、真实材质用照片、手作感来自不规则、克制、隐藏交互不提示、静止时零重画……
  - `feedback.md`：把「太抢眼」「塑料感」「太规整」「不是让你……」这类口语翻成具体动作。
  - `materials.md`：用图像模型生成白底 / 绿幕照片、抠图、局部重上色、导出和防缓存。
  - `performance.md`：性能规则和踩过的坑（症状 → 原因 → 修法）。
  - `verify.md`：Playwright 路由替身截图、量化检查、逐帧抖动检测、主线程占用。
- **风格层**（`styles/`）：一个风格一个包，可替换。
  - `journal-atelier.md`：手帐 / 拼贴 / 手作工坊风（撕边纸、和纸胶带、珍珠纽扣、麻绳、火漆、古董剪刀、缎布）。
  - `_template.md`：新风格包模板。换风格时只写一个新包，通用层不动。

## 安装

把整个目录放进 skills 目录即可：

```bash
git clone https://github.com/hewxs22-oss/atelier-ui.git ~/.claude/skills/atelier-ui
```

然后在 Claude Code 里用 `/atelier-ui [风格包] [页面]` 调用，或者直接说「把这个页面做得有质感一点」。

## 说明

- 文中的例子和文件路径（`tools/review_gloss.html`、`docs/review-home-style-prompt.md`、`knowledge/sessions/…`、`config.get_302ai_image_key()`）来自它的源项目，换到你的项目时替换成你自己的页面、风格文档和取密钥的函数。
- 图像模型的密钥一律通过项目自己的配置函数取，不要写进代码或输出。
- 正文是中文。

## License

MIT
