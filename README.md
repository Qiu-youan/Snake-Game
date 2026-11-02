# 贪吃蛇 · Snake

一个用 **HTML5 Canvas** 写的网页版贪吃蛇，整个游戏只有 `index.html` 一个文件，不需要安装依赖、不需要构建、也不加载任何外部图片或音频文件——所有图形都用 Canvas 绘制，所有音效都用 WebAudio 实时合成。

---

## 如何运行

### 方式一：直接双击（最简单）

用浏览器打开 `index.html` 即可：

- Windows：双击 `index.html`（或右键 → 打开方式 → Edge / Chrome / Firefox）
- macOS：`open index.html`
- 也可以把文件直接拖进浏览器窗口

游戏不依赖网络和本地服务器，`file://` 协议下即可完整运行。

### 方式二：起一个本地静态服务器（可选）

如果浏览器对 `file://` 有额外限制（个别企业策略会禁用 WebAudio），可以起一个静态服务器再访问：

```bash
# Python 3（系统自带即可）
cd TestGame
python -m http.server 8000
# 然后浏览器打开 http://localhost:8000
```

```bash
# 或者用 Node.js
npx serve .
```

### 浏览器要求

- 支持 Canvas 2D 的现代浏览器：Chrome / Edge / Firefox / Safari（含移动端）较新版本
- 用到 `canvas.roundRect()`，不支持时会自动降级为 `arcTo` 手工绘制圆角，不影响运行
- WebAudio 在部分浏览器里需要用户先与页面交互（按键或点击）才会出声，这是浏览器策略，点一下「开始游戏」即可解锁
- `localStorage` 用于保存最高分与设置；在隐私模式下写入失败会被静默忽略，游戏照常运行

---

## 怎么玩

| 操作 | 按键 |
| --- | --- |
| 移动 | 方向键 `↑ ↓ ← →` 或 `W A S D` |
| 暂停 / 继续 | `空格` 或 `P` |
| 重新开始 | `R` |
| 开始游戏 | `空格` / `Enter` / 点击按钮 |

手机端：

- 在游戏区域**滑动**改变方向
- 也可以使用屏幕下方的 `▲ ◀ ▼ ▶` 方向键（触屏设备才会显示）

### 规则

- 每吃一颗红色果实 **+10 分**，蛇身变长
- 每吃 **5 颗**升一档速度：每格移动耗时从 150ms 逐步加快到最快 72ms，HUD 的「速度」显示当前档位
- 吃到果实后有 **28%** 概率刷新**金色奖励果**，价值 **50 分**，带 8 秒倒计时圆环，超时会消失
- 撞到墙壁或自己的身体即游戏结束
- 可以切换**「穿墙」**模式：开启后从一侧穿出会从对侧进入，墙壁不再致命
- 最高分、静音状态、穿墙开关会保存在浏览器本地，下次打开仍然有效

---

## 用了什么技术

全部是浏览器原生能力，**零依赖、零构建、零外部资源**。

| 技术 | 用途 |
| --- | --- |
| **HTML5 Canvas 2D** | 游戏画面全部绘制：网格背景、蛇身、果实、粒子、光晕 |
| **原生 JavaScript（ES5 语法，IIFE 封装）** | 游戏逻辑，无需框架和打包工具 |
| **CSS3**（内联在 `<style>`） | 页面布局、HUD、霓虹风格按钮、遮罩层、响应式适配 |
| **`requestAnimationFrame`** | 渲染主循环，约 60fps |
| **固定步长累加器** | 逻辑更新与渲染解耦：`acc += dt`，累计到 `stepMs` 才走一格，保证不同刷新率下速度一致 |
| **WebAudio API** | 用 `OscillatorNode` + `GainNode` 实时合成音效（吃食物、升级、死亡），无音频文件 |
| **localStorage** | 持久化最高分、静音、穿墙设置 |
| **Pointer Events + `setPointerCapture`** | 鼠标与触屏统一的滑动操作 |
| **`devicePixelRatio` 缩放** | 高分屏下按物理像素渲染，画面不糊 |

### 代码结构（都在 `index.html` 里）

```
index.html
├── <style>            页面样式：深色霓虹主题、HUD、遮罩、方向键盘、响应式
├── <body>             结构：HUD（得分/最高/速度）+ 画布 + 暂停/结束遮罩 + 工具栏 + 方向键
└── <script>           游戏实现（约 680 行，IIFE 作用域）
    ├── 常量与状态      网格 24×24、单元格 24px、速度档位、菜单/游戏中/暂停/结束四态
    ├── 音效             ensureAudio / beep —— WebAudio 振荡器合成
    ├── 画布适配         resizeCanvas —— DPR 缩放
    ├── 食物与奖励        spawnFood / spawnBonus —— 从空闲格子中随机取位
    ├── 游戏流程         startGame / togglePause / gameOver / reset
    ├── 核心逻辑         stepGame —— 方向缓冲、撞墙判定、自撞判定、吃食物增长
    ├── 粒子系统         burst / updateParticles —— 吃到食物的爆散与死亡震屏
    ├── 渲染             drawBackground / drawFood / drawBonus / drawSnake / drawParticles
    ├── 主循环           loop —— 固定步长累加 + 每帧重绘
    └── 输入            键盘、滑动、屏幕方向键、工具栏按钮
```

### 几个实现细节

- **不会「180° 掉头」**：方向输入先进入长度上限为 2 的队列，入队时与队列最后一个方向比较，反方向直接丢弃；真正切换发生在每一格移动开始时，所以快速连按两下也能正确转弯。
- **自撞判定的边界情况**：如果这一步会吃到食物，尾巴不会移开，因此这种情形下需要把尾格也算进碰撞检测（`limit = eats ? snake.length : snake.length - 1`）。
- **奖励果的计时**：用只在游戏中累加的 `elapsed` 计算过期时间，暂停时倒计时不会偷跑。
- **画面抖动**：死亡时 `shake` 衰减，渲染前对画布做随机平移，并且背景多画一圈，避免抖动时露出边缘。
- **性能**：每段蛇身单独 `save/restore`，只有蛇头和食物使用 `shadowBlur` 光晕，粒子数量上限 260。

---

## 文件

```
TestGame/
├── index.html   # 游戏本体（HTML + CSS + JS 全部内联）
└── README.md    # 本文档
```

## 可以怎么改

- **改难度**：脚本开头的 `BASE_STEP`（初始速度，越小越快）、`MIN_STEP`（速度上限）、`GROW_PER_LEVEL`（几颗果实升一档）
- **改棋盘**：`COLS` / `ROWS` / `CELL`，注意画布尺寸 `W = COLS * CELL` 会随之变化
- **改配色**：CSS 变量 `--neon` / `--neon2` 控制界面主色；蛇身颜色由 `drawSnake` 里的 HSL 插值决定；果实颜色在 `drawFood` / `drawBonus` 中
- **改概率**：奖励果刷新概率在 `onEatFood` 里的 `Math.random() < 0.28`
