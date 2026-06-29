# 单标签页内导航 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让京东自动评价闭环始终在单个浏览器标签页内完成，消除挂机批量评价时累积几十个标签页导致的爆内存。

**Architecture:** 三个跳转步骤（列表→评价、评价→成功、成功→列表）全部经过唯一的导航点击入口 `autoClickAfter()`。在该函数点击前，先用 `forceSameTabNav()` 把目标元素及祖先链上 `<a>`/`<form>` 的 `target` 强制设 `_self`（阻止 `target="_blank"` 新开页），再用 `withOpenGuard()` 在点击瞬间临时把 `window.open` 改为原地 `location.href` 跳转（兜住 onclick 里的程序化 `window.open`），点完立即还原。异常时退回普通 `click()`，最坏情况等同现状。一处改动覆盖三处跳转。

**Tech Stack:** Tampermonkey 用户脚本（原生 JS + jQuery，单文件 IIFE `jd-auto-review.user.js`）。无构建系统、无包管理器、无测试框架、无 linter。

**Testing note（重要）：** 本仓库无任何自动化测试。TDD 的"写失败测试 → 跑测试"循环在此替换为两层验证：
- **算法自检**：把两个纯函数的源码复制到任意标签页的 DevTools Console 运行断言（无需京东、无需登录），快速验证逻辑正确。
- **真机验证**：把脚本粘进 Tampermonkey，在真实登录的京东页面跑闭环，观察标签页数量不变。
不要尝试引入 `npm`/`vitest`/`jsdom` 等测试基建——这违背仓库现状且超出本任务范围。

---

## File Structure

只动两个文件：

- **Modify** `jd-auto-review.user.js`（单文件 IIFE）
  - 新增模块级函数 `forceSameTabNav(el)`、`withOpenGuard(fn)`，插入位置：`$clickableByText` 函数之后、`autoClickAfter` 函数之前（当前约 `jd-auto-review.user.js:138` 与 `:141` 之间）。
  - 修改 `autoClickAfter` 内的点击块（当前约 `:156-164`），接入上述两个函数 + `try/catch` 降级。
  - 修改元信息 `@version 8.5` → `@version 8.6`（第 4 行）。
- **Modify** `README.md`：在「已知限制与风险」补一条标签页说明。

不新建任何文件。两个新函数与现有代码同在一个 IIFE 作用域，靠 JS 函数声明提升，定义位置在调用处之前即可。

---

## Task 1: 新增 `forceSameTabNav` 与 `withOpenGuard` 两个工具函数

**Files:**
- Modify: `jd-auto-review.user.js`（在 `$clickableByText` 之后、`autoClickAfter` 之前插入）

- [ ] **Step 1: 在 `jd-auto-review.user.js` 插入两个函数**

定位 `$clickableByText` 函数结尾（其 `}` 闭合后、`// 通用：倒计时后自动点击...` 注释 / `function autoClickAfter` 之前），插入：

```js
    // 强制单标签页导航：把点击目标及其祖先链上的 <a>/<form> 的 target 设为 _self，
    // 阻止京东 target="_blank" 链接新开标签页。el 为原生 DOM 元素。
    function forceSameTabNav(el) {
        let node = el;
        while (node && node !== document.body) {
            const tag = node.tagName;
            if (tag === 'A' || tag === 'FORM') {
                node.setAttribute('target', '_self');
            }
            node = node.parentNode;
        }
    }

    // 点击期间临时把 window.open 改成"原地跳转"，兜住京东 onclick 里
    // window.open(...) 程序化开新页。fn 跑完立即还原，并再加 500ms 保险还原（防延迟调用 / finally 漏跑）。
    function withOpenGuard(fn) {
        const orig = window.open;
        let restored = false;
        const restore = function() {
            if (restored) return;
            restored = true;
            window.open = orig;
        };
        window.open = function(url) {
            try { if (url) location.href = url; } catch (e) {}
            return null;
        };
        try {
            fn();
        } finally {
            restore();
            setTimeout(restore, 500);
        }
    }
```

- [ ] **Step 2: 控制台自检 `forceSameTabNav` 算法**

打开任意非京东标签页（如 `about:blank`），DevTools Console 粘贴并回车：

```js
function forceSameTabNav(el) {
    let node = el;
    while (node && node !== document.body) {
        const tag = node.tagName;
        if (tag === 'A' || tag === 'FORM') node.setAttribute('target', '_self');
        node = node.parentNode;
    }
}
// case 1: 点的就是 <a> 本身
const a1 = document.createElement('a'); a1.target = '_blank';
forceSameTabNav(a1);
console.assert(a1.target === '_self', 'case1 失败', a1.target);
// case 2: 点的是 <a> 内的 <span>
const a2 = document.createElement('a'); a2.target = '_blank';
const span = document.createElement('span'); a2.appendChild(span);
forceSameTabNav(span);
console.assert(a2.target === '_self', 'case2 失败', a2.target);
console.log('forceSameTabNav 自检通过');
```

Expected: 打印 `forceSameTabNav 自检通过`，**无** `Assertion failed` 报错。

- [ ] **Step 3: 控制台自检 `withOpenGuard` 算法**

同一 Console 继续粘贴：

```js
function withOpenGuard(fn) {
    const orig = window.open;
    let restored = false;
    const restore = function() { if (restored) return; restored = true; window.open = orig; };
    window.open = function(url) { try { if (url) location.href = url; } catch (e) {} return null; };
    try { fn(); } finally { restore(); setTimeout(restore, 500); }
}
const origOpen = window.open;
let during = null;
withOpenGuard(function() { during = (window.open !== origOpen); });
console.assert(during === true, '执行期间 window.open 应被接管');
console.assert(window.open === origOpen, '执行后应还原为原始 window.open');
console.log('withOpenGuard 自检通过：执行期接管 =', during);
```

Expected: 打印 `withOpenGuard 自检通过：执行期接管 = true`，**无** `Assertion failed`。这验证两条契约：执行 `fn` 期间 `window.open` 已被替换；`fn` 返回后已还原。

- [ ] **Step 4: 提交**

```bash
git add jd-auto-review.user.js
git commit -m "feat: 新增 forceSameTabNav / withOpenGuard，为单标签页导航做准备"
```

---

## Task 2: 接入 `autoClickAfter`（含 try/catch 降级）

**Files:**
- Modify: `jd-auto-review.user.js`（`autoClickAfter` 内点击块，约 `:156-164`）

- [ ] **Step 1: 修改点击块**

把 `autoClickAfter` 内当前的：

```js
            const $el = getEl();
            if ($el && $el.length > 0) {
                updateStatus(`${label}：已自动点击 ✅`, 'green');
                $el[0].click();
            } else {
                updateStatus(`${label}：未找到目标元素，流程已停止。`, 'red');
                if (onNotFound) onNotFound();
            }
```

改为：

```js
            const $el = getEl();
            if ($el && $el.length > 0) {
                updateStatus(`${label}：已自动点击 ✅`, 'green');
                // 单标签页导航：点击前中和 target="_blank"，并接管 window.open 兜底程序化开新页。
                // 中和失败则退回普通点击，最坏情况等同现状（开新标签页），不会更差。
                try {
                    forceSameTabNav($el[0]);
                    withOpenGuard(function() { $el[0].click(); });
                } catch (e) {
                    updateStatus('单标签导航中和失败，退回普通点击：' + e, 'red');
                    $el[0].click();
                }
            } else {
                updateStatus(`${label}：未找到目标元素，流程已停止。`, 'red');
                if (onNotFound) onNotFound();
            }
```

- [ ] **Step 2: 静态自检（通读 `autoClickAfter` 整个函数）**

确认：
1. `forceSameTabNav($el[0])` 在 `withOpenGuard(...)` **之前**调用（先去 target，再接管 open，最后点击）。
2. `try/catch` 包住两行调用；`catch` 内有普通 `$el[0].click()` 兜底。
3. `$el.length > 0` 的存在性检查仍在（保证 `$el[0]` 安全）。
4. `else` 分支（未找到元素 → `onNotFound`）未被改动。

- [ ] **Step 3: 安装到 Tampermonkey**

Tampermonkey → 管理面板 → 找到「京东自动评价（大模型版·全自动闭环）」→ 编辑 → 用 `jd-auto-review.user.js` 全文覆盖 → **填入 `API_KEY`** → 保存。（本地测试直接在 Tampermonkey 编辑器改即可，无需走 `*.local.user.js`。）

- [ ] **Step 4: 京东真机验证（核心验收）**

1. 登录京东，打开 `https://club.jd.com/myJdcomments/myJdcomment.action`（我的评价列表）。
2. 记下当前浏览器标签页数量 `N`。
3. 点浮窗「开始」。
4. 盯住一整轮 `列表 → 评价 → 成功 → 列表`：所有跳转都应在**当前标签页内**完成（地址栏 URL 变化、页面内容切换），**不得有新标签页出现**。
5. 连续观察 2–3 单（多轮循环），标签数始终 = `N`。
6. 三处跳转逐项确认不开新标签：列表页点「评价」、评价页点「发表」、成功页点「返回待评价列表」。
7. 旧功能不回归：评价回填、统一打五星、配图（`ENABLE_IMAGE=true` 时缩略图出现）、发表成功跳转、暂停/续作仍停在步骤边界，均正常。
8. DevTools Console 的 `[AI Auto Review]` 日志正常，**不应**出现红字「单标签导航中和失败」。

Expected: 标签数全程不变；Console 无中和失败日志；闭环正常推进。

- [ ] **Step 5: 提交**

```bash
git add jd-auto-review.user.js
git commit -m "feat: 跳转前中和 target=_blank 并接管 window.open，强制单标签页内导航"
```

---

## Task 3: 更新 README「已知限制与风险」

**Files:**
- Modify: `README.md`（「已知限制与风险」列表）

- [ ] **Step 1: 增加一条说明**

在 `README.md` 的「## 已知限制与风险」列表末尾（合规风险那条之前或之后均可）追加：

```markdown
- **标签页**：v8.6 起整条闭环在单个标签页内完成（跳转前中和 `target="_blank"`、接管 `window.open`），挂机批量评价不再累积标签页。但历史运行已堆出的旧标签页需手动关闭一次——浏览器禁止脚本关闭非脚本打开的标签页，无法代为回收。
```

- [ ] **Step 2: 提交**

```bash
git add README.md
git commit -m "docs: 已知限制补充 v8.6 单标签页导航与历史标签页需手动清理的说明"
```

---

## Task 4: 版本号 8.5 → 8.6 并准备 GreasyFork 同步

**Files:**
- Modify: `jd-auto-review.user.js`（第 4 行 `@version`）

- [ ] **Step 1: 改版本号**

把第 4 行：

```js
// @version      8.5
```

改为：

```js
// @version      8.6
```

- [ ] **Step 2: 提交（release 触发同步）**

```bash
git add jd-auto-review.user.js
git commit -m "release: 8.6 —— 单标签页内导航（中和 target=_blank + 接管 window.open），消除挂机累积标签页"
```

> 按 CLAUDE.md：`release: X.Y —— <desc>` 是 GreasyFork webhook 同步的刻意触发提交，单独成 commit，与前面的 `feat:`/`docs:` 分开。

- [ ] **Step 3: push 到 main 触发对外发布（⚠️ 对外动作，执行前与用户确认时机）**

```bash
git push origin main
```

**此步骤会把 v8.6 推送给所有 GreasyFork 安装用户（对外发布、不可撤销）。** 不要自动执行——先向用户确认：本地真机验证是否通过、是否现在就发布。用户确认后再 `push`。push 后可到 GreasyFork 脚本页确认版本已更新。

---

## Self-Review（写完后自检，已执行）

**1. Spec coverage（对照 spec 各节）：**
- 改动位置仅 `autoClickAfter` → Task 2。✓
- 新增 `forceSameTabNav` / `withOpenGuard` → Task 1。✓
- 改写点击块（forceSameTabNav + withOpenGuard + try/catch）→ Task 2 Step 1。✓
- 优雅降级（try/catch 退回普通 click）→ Task 2 Step 1 代码 + Step 2 自检项。✓
- 不加配置开关（始终开启）→ 未引入任何配置项，符合。✓
- 测试（算法自检 + 真机标签数）→ Task 1 Step 2/3、Task 2 Step 4。✓
- 发布（8.5→8.6、feat:/docs:/release: 分提、README 补说明）→ Task 3、Task 4。✓
- 历史标签页不自动回收（手动关一次）→ Task 3 README 条目。✓
- 无遗漏。

**2. Placeholder scan：** 无 TBD/TODO/"适当处理错误"等；每个代码步骤均含完整可运行代码；自检片段含真实断言。✓

**3. Type consistency：** 两个函数名 `forceSameTabNav` / `withOpenGuard` 在 Task 1 定义、Task 2 调用，拼写与签名一致（`forceSameTabNav(el)` 传 `$el[0]`、`withOpenGuard(fn)` 传匿名函数）。✓ 变量 `$el`、`label`、`onNotFound`、`getEl` 均沿用 `autoClickAfter` 既有签名。✓
