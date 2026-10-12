# 星弦 · Starstrings

> 一片你用手指弹奏的星河。三种模式 = 三种力场 = 三种音色。
> 所有光与声皆为实时计算 —— 全文件**零外部资源**，单文件、无依赖、可离线。

`index.html` · 27KB · 676 行 · 单文件

---

## 这是什么

一块 canvas 上的粒子场，三种模式各是一套不同的力场模型，同时配一套不同的音色：

| 模式 | 视觉 | 交互 | 音色 |
|---|---|---|---|
| **星河** `galaxy` | 倾斜椭圆轨道，越靠外转越快，形成螺旋臂 | 拖动＝推开粒子扰动星轨；点按＝爆出尘粒 | 火花（2–3 个错开的短 blip） |
| **星云** `nebula` | 粒子绕着各自的家位缓慢漂移 | 按住＝引力把周围粒子拽向指尖；松手回散 | 引力嗡鸣（三个失谐分音，松手淡出） |
| **涟漪** `ripple` | 粒子静卧成网格 | 点按＝投石入水，一圈波前向外推开粒子再精确归位 | 水滴（正弦下滑 + 打水噪声瞬态） |

背景是三个 CSS 模糊色块在缓慢漂移，叠一层极细扫描线和暗角。中央是渐变标题与实时时钟。

**设计取向**：不要"炫技的密集粒子"，要有**触感**。所以每个模式都有明确的输入-响应关系，且响应是可预期的（能按得准、能玩出花样）。

---

## 快速开始

直接用浏览器打开 `index.html` 即可，无需构建、无需服务器。

在 Minis 里预览：

```
minis://workspace/index.html
```

> **iOS 音频说明**：WebAudio 必须在用户手势回调里创建/`resume()`。代码把 AudioContext 的创建放在首次 `pointerdown` 内，所以**第一次点按就是开声的那一下**。右上角 🔊/🔇 开关（状态存 `localStorage`），静音时连 AudioContext 都不会创建。

---

## 架构

无模块系统、无打包，全部在一个 IIFE 里，按职责分块：

```
(() => {                        // 整个应用
  ├─ 精灵预渲染  sprite(hue)     // 5 张 64px 径向渐变贴图，按色相缓存
  ├─ build()                    // 建粒子/星尘/网格基准，尺寸变化时重建
  ├─ target(p, time)            // 三模式的「目标位」函数
  ├─ step(dt)                   // 物理步进（弹簧 → 指针力 → 阻尼 → 限速）
  ├─ draw()                     // 绘制（拖尾淡出 → 星尘 → 粒子 → 尘粒）
  ├─ frame(now)                 // rAF 主循环，dt 钳制 60ms
  ├─ 交互层 onDown/up + pointer 事件
  └─ AudioFX 模块               // WebAudio 合成（IIFE 内再套 IIFE）
})();
```

### 渲染管线的两个关键决定

**1. 拖尾用 `destination-out` 淡出，而不是黑色半透明覆盖**

```js
ctx.globalCompositeOperation = "destination-out";
ctx.fillStyle = "rgba(0,0,0,0.26)";
ctx.fillRect(0, 0, W, H);
ctx.globalCompositeOperation = "lighter";   // 粒子用叠加混合
```

这样 canvas 始终保持**透明**，背后的 CSS 星云才能透出来。若用黑色覆盖，几帧后画布就变成不透明的黑底，星云全被盖住。

**2. 发光贴图预渲染**

逐点 `createRadialGradient` 在移动端是灾难。改为启动时生成 5 张 64px 的径向渐变 sprite（按色相缓存），每帧只做 `drawImage`。

---

## 物理参数（当前实值）

### 通用

| 参数 | 值 | 说明 |
|---|---|---|
| 粒子数 `N` | `clamp(area/2600, 90, 620)` | `prefers-reduced-motion` 时上限降到 220 |
| 星尘数 | `clamp(area/9000, 60, 260)` | 无物理，只闪烁 |
| DPR | `min(devicePixelRatio, 2)` | 上限 2，移动端省一半填充率 |
| `dt` | 钳制 ≤ 60ms | 切回前台时防跳帧打飞粒子 |
| **`MAXV`** | **10 px/帧**（≈600px/s） | 速度上限，见「踩坑 #2」 |
| 弹簧刚度 | `0.0014 × gs × dt`（`gs` 0.4–1.5） | 归位速度 |
| 阻尼 | `pow(0.90, dt/16)` | 每 16ms 衰减 10% |

### 指针力

| 参数 | 值 |
|---|---|
| 作用半径 `R` | `min(W,H) × 0.42` |
| 星云半径 `PR` | `R × 1.8`（引力范围更大） |
| 排斥强度（星河） | `+1.2` |
| 吸引强度（星云） | `−1.8` |
| 衰减曲线 | `(1 − d/PR) × 强度 × dt/16` |

> `d < 1px` 时方向退化（`dx/d` 抖动），用粒子自身的极角 `cos(p.a)/sin(p.a)` 兜底。

### 星河模式

| 参数 | 值 |
|---|---|
| 椭圆半径 | `RX = W × 0.42`，`RY = H × 0.40` |
| 轨道倾角 | `−0.25 rad` |
| 归一化半径分布 | `pow(random, 0.62)`（外密内疏） |
| 角速度 `spin` | `0.00022 – 0.00082 /ms`（约 12% 反向，增加层次） |
| 呼吸 | `1 + 0.05 × sin(t × 0.0006 + ph)` |

> 半径按**屏幕宽高归一化**而非对角线。竖屏下用对角线会让约 30% 粒子跑出视口（实测），现在实测 100% 在屏内。

### 星云模式

| 参数 | 值 |
|---|---|
| 家位网格 | `cols = round(sqrt(N·W/H))`，`rows = N/cols` |
| 漂移振幅 | `46px` |
| 漂移频率 | `t × 0.00042 × wob`（`wob` 0.6–2.6） |

### 涟漪模式

| 参数 | 值 |
|---|---|
| 波前速度 | `0.50 px/ms` |
| 空间包络 | `exp(−rel² / (2×120²))` |
| 振荡项 | `cos(rel / 34)` |
| 振幅 | `84px` |
| 时间衰减 | `exp(−(t−t0) / 3200)` |
| 波存活 | 4000ms 后丢弃 |

> 实测：峰值位移 64px，波前从 R33 扩展到 R388，静止后**归位误差 0.0px**。

### 尘粒（点按爆发）

| 参数 | 值 |
|---|---|
| 数量 | 30（reduced 12） |
| 初速 | 1.2 – 8.5 px/帧 |
| 寿命 | 620 – 1500ms |
| 上限 | 420 个（超出丢最旧） |

### 视觉

| 参数 | 值 |
|---|---|
| 色相池 | `[205, 220, 265, 300, 40]` |
| 粒子尺寸 | 8 – 42px（贴图绘制尺寸，非半径） |
| 不透明度 | 0.28 – 1.0，叠加 `sin(t×0.0016 + ph)` 闪烁 |

---

## 音频合成图谱

零素材文件 —— 没有 `.mp3`、没有 `.wav`，连混响的脉冲响应都是算出来的。

### 信号链

```
                                      ┌───────────────┐
  note(gain) ──┬──▶[StereoPanner]──┬──▶│ master (0.5)  │──▶[lowpass 8200Hz Q0.4]──▶ destination
               │                   │   └───────────────┘
               │                   │
               │                   └──▶[wet gain]──▶[convolver]──▶[×0.3]──┘
               │                                        ▲
  noise(gain) ─┘                             程序生成的 IR
                                           (1.8s 双声道白噪声 × (1−i/n)^2.8)
```

- **混响不用素材**：`createConvolver()` + 程序生成 IR。双声道各生成一段噪声乘指数衰减，左右去相关 → 自然的立体声尾音。
- **噪声源复用**：启动时生成一段 1.2s 白噪声 buffer，每次用 `start(t, randomOffset)` 从随机位置取 0.05–0.09s，不必每次新建。
- **gain 不能为 0**：`exponentialRampToValueAtTime` 不接受 0，包络起点用 `0.0001`。

### 音阶映射：画面即乐器

```js
// 大调五声音阶（[0,2,4,7,9]），跨两个八度共 10 音
SCALE[k] = 220 × 2^(pentatonic[k % 5]/12 + floor(k/5))
```

- **音高 ← 触点 Y**（越靠上越高）
- **声像 ← 触点 X**（左右跟随手指）
- 五声音阶保证：**随机乱点也不会撞出不协和音程**

### 四个声音的参数

**`drop(xn, yn)` — 涟漪点按：水滴**
```
正弦  f×1.9 → f×0.6 (指数, 0.9s)   增益 0.24   湿 0.55
正弦  f×3.02            (0.16s)    增益 0.05   湿 0.30
噪声  带通 2400+xn×1800 (0.09s)    增益 0.05   湿 0.35    ← "打水"瞬态
f = SCALE[3..8] 按 (xn, yn) 选取
```

**`spark(xn, yn)` — 星河/星云点按：火花**
```
n = 2 + rand(0..1) 个 blip，逐个错开 20–55ms
每个:  正弦/三角交替, 时长 0.18–0.34s, 增益 0.14 − i×0.03, 尾音 ×0.94 下滑
前置:  噪声 带通 3200Hz (0.07s) 增益 0.035
f = SCALE[5..9]
```

**`blip()` — 模式切换：柔和双音**
```
SCALE[2] @ t       0.5s  增益 0.12  声像 −0.1
SCALE[5] @ t+0.06  0.6s  增益 0.10  声像 +0.1
```

**`droneStart(xn, yn)` — 星云按住：引力嗡鸣**
```
基频 base = 55 × 2^(round((1−yn)×3)/12)      ← 拖动可移调
分音 [1, 1.5, 2.006]   ← 2.006 故意失谐，产生 6‰ 拍频（缓慢的"呼吸"）
低通 260Hz → 620Hz (1.6s 线性扫描) Q=3       ← 按住越久越亮
增益 0 → 0.1 (0.5s 指数上升)
松手: 0.45s 指数淡出, 0.55s 后 stop()
```

### 限流

| 限制 | 值 | 理由 |
|---|---|---|
| 并发语音数 | ≤ 30 | 超出直接丢弃，防止连点堆积 |
| 最小间隔 | 40ms | 同帧连点 20 次实测只出 1 声；正常手速（60ms）连点全部正常 |

---

## 踩坑记录

以下四条都是**实际调试中撞到并修掉的**，不是假想问题。

### 🔴 #1 限流器用 `ctx.currentTime` 计时 → 第一次点按就被永久静音

**症状**：点击没有声音。用节点计数法测出 `drop` 生成了 **0 个振荡器**。

**原因**：AudioContext 未解禁时 `state === 'suspended'`，而 **`currentTime` 恒为 0**。限流写成 `if (now − lastAt < 0.04) return`，`lastAt` 初值 0 → 间隔 0 → 首帧即被吃掉；而由于 currentTime 不前进，**之后每一帧仍算 0，永久静音**。

**结论**：限流一律用 `performance.now()`（毫秒）+ `lastAt = -1e9` 初值。

```js
let lastAt = -1e9;
function spend() {
  const now = performance.now();
  if (voices > 30 || (now - lastAt) < 40) return false;
  lastAt = now; voices++;
  setTimeout(() => { voices = Math.max(0, voices - 1); }, 1500);
  return true;
}
```

### 🔴 #2 指针力当「速度冲量」用 → 粒子被弹飞 696px

**症状**：拖动时粒子瞬间飞出屏幕。

**原因**：写成 `vx += f`（f≈132）叠加 0.9/帧阻尼。阻尼下冲量的总位移是 `f/(1−0.9)` ≈ **1320px**，实测单次 696px。

**结论**：指针力必须是**加速度**量级（每帧 1–2 像素），并加硬性限速：

```js
const f = (1 - d / PR) * (±1.2|1.8) * dt / 16;   // 不是 ×132
p.vx += ux * f;
// ... 阻尼后
const sp = Math.hypot(p.vx, p.vy);
if (sp > MAXV) { p.vx = p.vx / sp * MAXV; p.vy = p.vy / sp * MAXV; }
```

### 🟡 #3 入场动画的 `fill-mode`：`both` 与 `forwards` 都不完美

**症状**：内建浏览器截图里标题 `opacity: 0`，看不见。

**原因**：标签页 `document.hidden === true` 时 **CSS 动画不推进**。`both` 的向后填充让元素冻结在 `from` 状态（opacity 0）。

**权衡**（这是真正的取舍，不是 bug）：
- `both`：真实可见时动画立即开始，**无闪现** ✔；但「从未可见」时冻结在 0。
- `forwards`：不冻结 ✔；但**带 delay 的元素在延迟期显示自身样式 → 可见页面首屏闪现** ✘。

**结论**：保留 `both`（真实用户的体验优先）。验证截图时临时 `el.style.animation = 'none'`。

### 🟡 #4 星轨半径用对角线 → 竖屏下 30% 粒子跑出视口

**原因**：`maxR = hypot(W,H) × 0.52`。390×844 的竖屏上这是 483px，而半对角线才 465px、半宽只有 195px。

**结论**：改用**按屏幕宽高归一化的椭圆**，星系铺满视口：

```js
const ex = Math.cos(ang) * r * W * 0.42;
const ey = Math.sin(ang) * r * H * 0.40;
```

实测：在屏内粒子占比从 ~70% → **100%**。

### 🟡 #5 后台标签页 rAF 暂停 → canvas 全空

**原因**：`document.hidden` 时 `requestAnimationFrame` 不回调，画布从未绘制过。

**结论**：启动时同步画一帧，并在重新可见时补帧：

```js
resize();
step(16); draw();                 // 同步首帧，不依赖 rAF
requestAnimationFrame(frame);
document.addEventListener("visibilitychange", () => {
  if (!document.hidden) { last = performance.now(); step(16); draw(); }
});
```

> **推论（验证动效页时的第一条军规）**：截图里画面空白，先 `execute_js` 查 `document.hidden`，再怀疑代码。

---

## 自测方法（可复用）

这个项目**没有自动化测试**，但有两套在无头/静音环境下依然有效的验证手法。核心思路：**把主文件复制一份临时副本、注入钩子、跑断言、删掉副本** —— 不污染成品文件。

### 手法 A：物理钩子 —— 手动推进时间

问题：`document.hidden` 时 rAF 暂停，物理完全不跑，更没法测。解决：把 `step`/`draw`/`time` 暴露出来，手动 `advance + step`。

```python
# 用 python 精确注入，避免手改出错
s = open('index.html', encoding='utf-8').read()
hook = """  window.__t = {
    get particles(){return particles}, get waves(){return waves}, get motes(){return motes},
    step, draw, setMode2(m){mode=m; waves.length=0}, advance(ms){t+=ms},
    pointer:()=>pointer, get size(){return [W,H]}, onDown,
    release(){pointer.active=false;pointer.down=false;}
  };
"""
old = "  resize();\n  // 先同步画一帧"
assert old in s
open('_selftest.html','w',encoding='utf-8').write(s.replace(old, hook+old, 1))
```

然后就能断言：

```js
const T = window.__t, S = n => { for (let i=0;i<n;i++){ T.advance(16); T.step(16); } };
T.setMode2('galaxy'); S(300);
// → 可见粒子占比、最大半径
T.setMode2('ripple'); T.release(); S(400);
T.onDown(W/2, H/2); T.release(); S(10);
// → 波前位置、峰值位移、静止后归位误差
```

**实测记录**

| 检查项 | 结果 |
|---|---|
| NaN / 非有限值 | 0 |
| 速度上限 | 6.05 ≤ MAXV(10) ✔ |
| 星轨在屏内占比 | 100% |
| 涟漪峰值位移 | 64px @ 320ms |
| 涟漪静止归位误差 | **0.0px** |
| 尘粒 200 帧后清空 | 30 → 0 ✔ |

> ⚠️ **测试陷阱**：`onDown` 会把指针激活，若模拟完后不调用 `release()`，指针斥力会**持续作用**，让你误判"粒子归不了位"。

### 手法 B：AudioContext 代理 —— 在听不到声音时验证音频

在主脚本**之前**插一段代理，统计各类节点的创建次数：

```html
<script>
window.__calls = {osc:0, buf:0, conv:0, panner:0};
window.__errs = [];
window.addEventListener('error', e => window.__errs.push(String(e.message)));
(function () {
  const AC = window.AudioContext || window.webkitAudioContext; if (!AC) return;
  function W(a) { const c = new AC(a);
    const o = c.createOscillator.bind(c), b = c.createBufferSource.bind(c);
    c.createOscillator    = function(){ window.__calls.osc++; return o(); };
    c.createBufferSource  = function(){ window.__calls.buf++; return b(); };
    if (c.createConvolver)  { const v=c.createConvolver.bind(c);   c.createConvolver  = function(){ window.__calls.conv++;   return v(); }; }
    if (c.createStereoPanner){const v=c.createStereoPanner.bind(c);c.createStereoPanner=function(){ window.__calls.panner++; return v(); }; }
    return c;
  }
  W.prototype = AC.prototype;
  window.AudioContext = W;
  if (window.webkitAudioContext) window.webkitAudioContext = W;
})();
</script>
```

再用合成事件触发各路径，对比计数增量：

```js
const fire = (t,x,y) => document.getElementById('stage')
  .dispatchEvent(new PointerEvent(t, {clientX:x, clientY:y, bubbles:true, pointerId:1}));
```

**实测记录**

| 场景 | 生成节点 | 判定 |
|---|---|---|
| 首次点按（涟漪） | conv 1 + osc 2 + buf 1 | 第一击有声 ✔ |
| 星河点按 | osc 2–3 + buf 1 | 火花簇 ✔ |
| 星云按住 | osc 3 | 三音嗡鸣 ✔ |
| 松手 | 0（仅包络淡出） | ✔ |
| 按住时切模式 | 嗡鸣掐断 + 切换音 | ✔ |
| **静音状态点按** | **0** | 完全静默 ✔ |
| 同帧连点 20 次 | 2（仅 1 声） | 限流生效 ✔ |
| 间隔 60ms 连点 20 次 | 40 | 正常连击不被误杀 ✔ |

`window.__errs` 为空 —— 所有路径无异常。

---

## 调参指南

想改什么，改哪里：

| 想要的效果 | 改这里 |
|---|---|
| 粒子更多/更少 | `build()` 里 `area / 2600` 的除数 |
| 粒子更大/更小 | `size: rnd(8, 42)` |
| 星轨转更快 | `spin: rnd(0.00022, 0.00082)` |
| 星轨更扁/更圆 | `target()` 里 `W * 0.42` / `H * 0.40` 的比例 |
| 拖动更"顶手" | 指针力 `? -1.8 : 1.2` 里的 `1.2`（或抬高 `MAXV`） |
| 引力更霸道 | `2` 那个 `-1.8` 或 `R * 1.8` |
| 涟漪更大/更慢 | `* 84`（振幅）、`* 0.50`（波速）、`/ 34`（波长） |
| 粒子回位更快 | 弹簧 `0.0014`；回弹太跳就调阻尼 `0.90` |
| 换配色 | `HUES = [205, 220, 265, 300, 40]` |
| 声音更大/更小 | `master.gain.value = 0.5` |
| 混响更长 | `makeIR(ctx, 1.8, 2.8)` → `(3.0, 2.2)` |
| 换音阶 | `SCALE` 里的 `[0, 2, 4, 7, 9]`（如改小调 `[0,3,5,7,10]`） |
| 水滴更低沉 | `drop()` 里的 `SCALE[3..8]` → `SCALE[0..5]` 或降低基频 220 |
| 嗡鸣更厚 | `droneStart()` 的 `[1, 1.5, 2.006]` 分音表 |
| 允许更密的连击 | `spend()` 里的 `40`（毫秒） |

---

## 无障碍与兼容

- `prefers-reduced-motion: reduce`：粒子上限 620 → 220，背景色块动画关闭，闪烁提示关闭
- 安全区适配：`env(safe-area-inset-*)`（顶部标题、底部按钮组）
- `touch-action: none` + `user-select: none`：防止拖拽时页面滚动/选中
- canvas 缩放上限 `DPR ≤ 2`：移动端省一半填充率
- 若 `createStereoPanner` 不存在（老 Safari），自动降级为纯单声道，不报错
- 若 `AudioContext` 不存在，所有音效调用静默返回 `null`，视觉不受影响

---

## 文件清单

```
shared/starstrings/
└── README.md            # 本文档

workspace/
└── index.html           # 星弦本体（27KB 单文件，零外部资源）
```

`.gitignore` 建议：临时副本 `_selftest.html` / `_audiotest.html` 是调试产物，**不要提交**。

---

## 变更记录

**v1.0** — 初版
- 三模式粒子物理（轨道椭圆 / 引力漂移 / 网格涟漪）
- WebAudio 实时合成音效（水滴 / 火花 / 切换音 / 引力嗡鸣）
- 程序生成混响脉冲响应，零素材文件
- 修掉：限流器致命静音 bug、指针力冲量失控、竖屏粒子越界、后台标签页空白画布
