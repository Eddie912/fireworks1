# 粒子汇聚爱心动画实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在单个 `index.html` 中实现无需依赖、可直接打开的粒子汇聚爱心动画。

**Architecture:** 页面使用一个全屏 Canvas 2D 绘制全部粒子，CSS 只负责视口与降级界面。JavaScript 将爱心采样、粒子状态、渲染循环、交互和响应式适配拆成独立函数，并只保留一个 `requestAnimationFrame` 循环。

**Tech Stack:** HTML5、CSS3、原生 JavaScript、Canvas 2D、Node.js 内置断言、Google Chrome Headless、Playwright Core。

## Global Constraints

- 唯一交付文件为 `index.html`，CSS 和 JavaScript 必须内嵌。
- 不引入第三方运行时依赖、网络请求、音频或外部媒体。
- 桌面粒子目标约 700 个，移动端约 420 个，按视口面积动态调整。
- 使用设备像素比渲染，倍率最大值固定为 2。
- 必须支持鼠标、触摸、窗口缩放和 `prefers-reduced-motion: reduce`。
- Canvas 必须设置 `aria-hidden="true"`，并另外提供屏幕阅读器文本说明。
- 页面不得出现滚动条、空白画面或重复动画循环。

---

### Task 1: 创建页面骨架与视觉基底

**Files:**
- Create: `index.html`

**Interfaces:**
- Consumes: 无。
- Produces: `#heartCanvas` Canvas、`#fallback` 降级提示、`#description` 屏幕阅读器说明、`.hint` 交互提示。

- [ ] **Step 1: 写入失败的结构检查**

Run:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');for(const s of ['id=\"heartCanvas\"','id=\"fallback\"','id=\"description\"','aria-hidden=\"true\"'])if(!h.includes(s))throw new Error('missing '+s)"
```

Expected: FAIL，因为 `index.html` 尚不存在。

- [ ] **Step 2: 创建最小 HTML 与视觉基底**

使用 `apply_patch` 创建 `index.html`，至少包含以下结构和样式，并在文件末尾预留单独的 `<script>` 区域：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
  <title>粒子心跳</title>
  <style>
    :root { color-scheme: dark; }
    * { box-sizing: border-box; }
    html, body { width: 100%; height: 100%; margin: 0; overflow: hidden; }
    body {
      background:
        radial-gradient(circle at 50% 48%, rgba(174, 27, 77, .24), transparent 32%),
        radial-gradient(circle at 14% 18%, rgba(63, 20, 92, .22), transparent 38%),
        linear-gradient(145deg, #05060d 0%, #100617 54%, #04050b 100%);
      font-family: ui-sans-serif, system-ui, sans-serif;
    }
    #heartCanvas { position: fixed; inset: 0; width: 100%; height: 100%; }
    .hint {
      position: fixed; left: 50%; bottom: max(24px, env(safe-area-inset-bottom));
      margin: 0; transform: translateX(-50%); color: rgba(255, 224, 235, .62);
      font-size: 12px; letter-spacing: .16em; white-space: nowrap; user-select: none;
    }
    #fallback {
      position: fixed; inset: 50% auto auto 50%; width: min(86vw, 420px);
      margin: 0; transform: translate(-50%, -50%); color: #ffe5ef;
      line-height: 1.7; text-align: center;
    }
    .sr-only {
      position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
      overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
    }
  </style>
</head>
<body>
  <canvas id="heartCanvas" aria-hidden="true"></canvas>
  <p class="hint" aria-hidden="true">移动或点击，让心跳回应你</p>
  <p id="fallback" hidden>当前浏览器无法显示爱心动画。</p>
  <p id="description" class="sr-only">装饰性粒子爱心动画。画面中的粒子会汇聚成爱心并轻柔跳动。</p>
  <script></script>
</body>
</html>
```

- [ ] **Step 3: 验证结构检查通过**

Run:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');for(const s of ['id=\"heartCanvas\"','id=\"fallback\"','id=\"description\"','aria-hidden=\"true\"'])if(!h.includes(s))throw new Error('missing '+s);console.log('structure: ok')"
```

Expected: PASS，并输出 `structure: ok`。

- [ ] **Step 4: 验证页面可被本地静态服务器加载**

Run:

```bash
python3 -m http.server 4173
```

在另一个终端运行：

```bash
curl -sSf http://127.0.0.1:4173/index.html | rg 'id="heartCanvas"'
```

Expected: 输出包含 `id="heartCanvas"` 的 HTML 行；完成后停止服务器。

- [ ] **Step 5: 提交骨架**

```bash
git add index.html
git commit -m "feat: add heart animation page shell"
```

### Task 2: 实现爱心采样、粒子状态与基础动画

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `#heartCanvas`、`#fallback`。
- Produces: `window.__heartAnimation.createHeartPoints(width, height, count)`、`window.__heartAnimation.createParticles(points, width, height)`、`window.__heartAnimation.updateParticle(particle, time)`。

- [ ] **Step 1: 写入失败的粒子 API 检查**

Run:

```bash
node - <<'NODE'
const fs = require('fs');
const vm = require('vm');
const assert = require('assert');
const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script>([\s\S]*?)<\/script>/);
assert(match, 'inline script missing');
const context = {
  window: {},
  document: { readyState: 'loading', getElementById: () => null, addEventListener: () => {} },
  requestAnimationFrame: () => 1,
  cancelAnimationFrame: () => {},
  Math,
};
vm.runInNewContext(match[1], context);
const api = context.window.__heartAnimation;
assert(api, 'debug API missing');
const points = api.createHeartPoints(1000, 800, 240);
assert.equal(points.length, 240);
assert(points.every((point) => point.x >= 0 && point.x <= 1000 && point.y >= 0 && point.y <= 800), 'points out of bounds');
const particles = api.createParticles(points, 1000, 800);
assert.equal(particles.length, points.length);
assert(particles.every((particle, index) => particle.tx === points[index].x && particle.ty === points[index].y));
console.log('particle api: ok');
NODE
```

Expected: FAIL，因为脚本区域为空，`debug API missing`。

- [ ] **Step 2: 实现爱心采样与粒子初始化**

在 `<script>` 中加入严格模式 IIFE，并实现以下纯函数：

```js
function createHeartPoints(width, height, count = 700) {
  const scale = Math.min(width * 0.34, height * 0.38, 360) / 16;
  const cx = width / 2;
  const cy = height / 2;
  return Array.from({ length: count }, (_, index) => {
    const t = index / count * Math.PI * 2;
    const outer = index % 5 < 3;
    const radius = outer ? 1 : Math.sqrt(Math.random()) * 0.86;
    const hx = 16 * Math.sin(t) ** 3;
    const hy = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);
    return { x: cx + hx * scale * radius, y: cy - hy * scale * radius };
  });
}

function createParticles(points, width, height) {
  return points.map((point, index) => ({
    x: Math.random() * width,
    y: Math.random() * height,
    tx: point.x,
    ty: point.y,
    vx: 0,
    vy: 0,
    phase: Math.random() * Math.PI * 2,
    speed: 0.026 + Math.random() * 0.032,
    size: 0.7 + Math.random() * 1.7,
    color: ['255,91,145', '255,128,151', '255,208,220', '255,176,103'][index % 4],
  }));
}
```

- [ ] **Step 3: 实现粒子更新与 Canvas 渲染**

加入 Canvas 获取、状态对象、`resizeCanvas()`、`updateParticle()` 和 `render()`。初始化时：

```js
const canvas = document.getElementById('heartCanvas');
const fallback = document.getElementById('fallback');
const context = canvas && canvas.getContext('2d');

if (!context) {
  canvas?.setAttribute('hidden', '');
  fallback?.removeAttribute('hidden');
} else {
  resizeCanvas();
  requestAnimationFrame(render);
}

window.__heartAnimation = { createHeartPoints, createParticles, updateParticle, getState: () => state };
```

`resizeCanvas()` 必须按以下规则计算：

```js
const dpr = Math.min(window.devicePixelRatio || 1, 2);
const rect = canvas.getBoundingClientRect();
state.width = rect.width;
state.height = rect.height;
canvas.width = Math.round(rect.width * dpr);
canvas.height = Math.round(rect.height * dpr);
context.setTransform(dpr, 0, 0, dpr, 0, 0);
const count = Math.round(Math.max(220, Math.min(700, rect.width * rect.height / 1750)));
const points = createHeartPoints(rect.width, rect.height, count);
state.particles = createParticles(points, rect.width, rect.height);
```

`updateParticle()` 使用 `1 - Math.pow(0.96, delta)` 形式的缓动、轻微正弦漂移和速度衰减；`render()` 每帧清空 Canvas、调用更新函数，并用 `globalCompositeOperation = 'lighter'` 绘制发光粒子。

- [ ] **Step 4: 验证纯函数与语法**

重新运行 Step 1 的检查，并执行：

```bash
node -e "const fs=require('fs'),vm=require('vm'),h=fs.readFileSync('index.html','utf8'),s=h.match(/<script>([\s\S]*?)<\/script>/)[1];new vm.Script(s);console.log('syntax: ok')"
```

Expected: `particle api: ok` 与 `syntax: ok`。

- [ ] **Step 5: 进行首次浏览器渲染检查**

启动 `python3 -m http.server 4173` 后运行：

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu --hide-scrollbars --window-size=1440,900 --virtual-time-budget=5000 --screenshot=/tmp/heart-basic.png http://127.0.0.1:4173/index.html
```

Expected: 生成 `/tmp/heart-basic.png`，中央可见由发光粒子组成的爱心，页面无错误页和空白区域。使用图像查看工具检查截图。

- [ ] **Step 6: 提交基础动画**

```bash
git add index.html
git commit -m "feat: render animated particle heart"
```

### Task 3: 完成交互、响应式与无障碍行为

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `createHeartPoints()`、`createParticles()`、`updateParticle()`、`render()`、`resizeCanvas()`。
- Produces: `window.__heartAnimation.burstAt(x, y)`、指针扰动、窗口缩放适配、减少动态效果降级。

- [ ] **Step 1: 写入失败的行为契约检查**

Run:

```bash
node - <<'NODE'
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
for (const token of ['pointermove', 'pointerdown', 'resizeCanvas', 'prefers-reduced-motion', 'burstAt']) {
  if (!html.includes(token)) throw new Error('missing ' + token);
}
console.log('behavior contract: ok');
NODE
```

Expected: FAIL，首个缺失标记为 `pointermove`。

- [ ] **Step 2: 实现指针扰动与触控扩散**

添加一个 passive `pointermove` 监听器：对 120 像素半径内粒子增加径向速度，强度使用 `(1 - distance / 120) * 2.4`。添加 `pointerdown` 监听器调用：

```js
function burstAt(x, y) {
  state.pointer = { x, y, active: true, strength: 1 };
  state.burstStart = performance.now();
  state.particles.forEach((particle) => {
    const dx = particle.x - x;
    const dy = particle.y - y;
    const distance = Math.hypot(dx, dy) || 1;
    const impulse = 4.5 + Math.random() * 5.5;
    particle.vx += dx / distance * impulse;
    particle.vy += dy / distance * impulse;
  });
}
```

渲染时让 `state.burstStart` 对应的扩散冲量在约 900 毫秒内衰减，并确保粒子随后重新汇聚。

- [ ] **Step 3: 实现缩放与减少动态效果**

- 监听 `resize`，通过单次 `requestAnimationFrame` 合并连续事件后调用 `resizeCanvas()`。
- 使用 `window.matchMedia('(prefers-reduced-motion: reduce)')` 读取初始偏好并监听变化。
- 减少动态效果开启时，只绘制一个无漂移的呼吸帧，不启动持续渲染循环，不响应爆炸扩散。
- 关闭减少动态效果后恢复单循环动画，且不能叠加多个循环。

- [ ] **Step 4: 验证行为契约与语法**

Run:

```bash
node - <<'NODE'
const fs = require('fs');
const vm = require('vm');
const html = fs.readFileSync('index.html', 'utf8');
for (const token of ['pointermove', 'pointerdown', 'resizeCanvas', 'prefers-reduced-motion', 'burstAt']) {
  if (!html.includes(token)) throw new Error('missing ' + token);
}
new vm.Script(html.match(/<script>([\s\S]*?)<\/script>/)[1]);
console.log('behavior contract: ok');
console.log('syntax: ok');
NODE
```

Expected: `behavior contract: ok` 与 `syntax: ok`。

- [ ] **Step 5: 使用 Playwright Core 检查桌面和移动端渲染**

启动 `python3 -m http.server 4173` 后运行内联 Playwright 脚本。脚本必须使用系统 Chrome 的 `executablePath`，执行以下检查：

```js
const results = [];
for (const viewport of [{ width: 1440, height: 900 }, { width: 375, height: 812 }]) {
  const page = await browser.newPage({ viewport });
  const errors = [];
  page.on('console', (message) => message.type() === 'error' && errors.push(message.text()));
  page.on('pageerror', (error) => errors.push(error.message));
  await page.goto('http://127.0.0.1:4173/index.html');
  await page.waitForTimeout(4500);
  await page.mouse.move(viewport.width * .62, viewport.height * .44);
  await page.mouse.click(viewport.width * .5, viewport.height * .5);
  await page.waitForTimeout(1200);
  const stats = await page.locator('#heartCanvas').evaluate((canvas) => {
    const ctx = canvas.getContext('2d');
    const data = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
    let lit = 0;
    let alpha = 0;
    for (let index = 3; index < data.length; index += 160) {
      alpha += data[index];
      if (data[index] > 20) lit += 1;
    }
    return { width: canvas.width, height: canvas.height, lit, alpha };
  });
  await page.screenshot({ path: `/tmp/heart-${viewport.width}.png` });
  results.push({ viewport, errors, stats });
  await page.close();
}
console.log(JSON.stringify(results, null, 2));
```

验收条件：

- 两个视口的 `errors` 都为空。
- `lit` 均大于 200，说明 Canvas 不是空白画面。
- 截图中的爱心居中、不被裁切，提示文字与爱心不重叠。
- 点击后动画继续运行且 Canvas 仍保持非空。

- [ ] **Step 6: 检查减少动态效果**

使用 Playwright 的 `emulateMedia({ reducedMotion: 'reduce' })` 再次打开页面，等待 500 毫秒后读取 Canvas 像素。预期 Canvas 非空，且没有持续大幅帧间差异。

- [ ] **Step 7: 执行最终静态检查并提交**

```bash
rg -n "https?://|script src=|TODO|TBD" index.html
git diff --check
git status --short
```

Expected: `rg` 无输出；`git diff --check` 无输出；`git status` 仅显示预期的 `index.html` 修改。随后执行：

```bash
git add index.html
git commit -m "feat: add interactive responsive heart animation"
```

## Self-Review

- **Spec coverage:** 交付形式、粒子汇聚、持续跳动、鼠标/触摸交互、点击扩散、响应式、粒子预算、设备像素比、减少动态效果、Canvas 降级和验收条件均在 Task 1 至 Task 3 中有对应步骤。
- **Placeholder scan:** 未使用 TBD、TODO、“稍后补充”或未定义占位函数；所有核心函数与状态均在任务接口中命名。
- **Type consistency:** `createHeartPoints()` 返回 `{ x, y }[]`；`createParticles()` 消费该结构并产出带 `tx/ty` 的粒子对象；`updateParticle()`、`render()`、`burstAt()` 均消费同一粒子结构。
