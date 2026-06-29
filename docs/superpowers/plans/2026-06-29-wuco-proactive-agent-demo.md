# WUCO Proactive Agent 网页 Demo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个可双击运行的纯前端网页 Demo，模拟元宝聊天界面，演示「会话内实时意图感知 → 触发评分 → 低打扰主动卡片」的完整闭环。

**Architecture:** 三层无构建依赖的纯 JS：`engine.js`（纯逻辑：意图分类、信号提取、触发评分——可在 Node 下单测）、`scenarios.js`（脚本化场景数据 + 状态机）、`ui.js`（DOM 渲染与事件）。`index.html` 用经典 `<script>` 标签按序加载三者（不使用 ES module，确保 `file://` 双击直接运行）。`engine.js`/`scenarios.js` 带 UMD 式尾部导出，使其同时能被浏览器全局与 Node `require` 使用，从而支持 TDD。

**Tech Stack:** HTML5 + CSS3 + 原生 ES5/ES6 JavaScript（零依赖、零构建）。测试用 Node 内置 `assert` + 一个 30 行的微型测试运行器（无 npm 依赖）。

## Global Constraints

- 纯前端，**双击 `index.html` 即可运行**，无后端、无 npm install、无构建步骤。
- 所有脚本用经典 `<script src>` 加载，**禁止** `type="module"` / `import` / `export`（`file://` 下会被 CORS 阻断）。
- `engine.js` 与 `scenarios.js` **必须无 DOM 依赖**，尾部用 `if (typeof module!=='undefined'&&module.exports) module.exports=...; else root.WUCO=...` 双模式导出。
- 浏览器侧全局命名空间统一为 `window.WUCO`（`WUCO.engine`、`WUCO.scenarios`）。
- 触发公式固定为：`score = demand×timeliness×confidence×acceptance×100 − interruptionCost×40 − λ×taskSwitchPenalty×100`，`λ=0.5`，各因子 clamp 到 0..1。
- 触发只锚定「意图爬升轨迹 + 任务边界」，**严禁 idle/定时触发**（设计原则，需体现在代码注释）。
- 文案语言为简体中文；主线场景=研究/写作型任务推进，黑客松自指=结尾彩蛋。
- 测试命令统一为 `node tests/<name>.test.js`，期望末行打印 `ALL PASS (<n>)`。
- 提交信息以 `feat:`/`test:`/`docs:`/`chore:` 前缀，结尾附 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。

---

## File Structure

| 文件 | 职责 |
|---|---|
| `index.html` | 单页骨架：左侧聊天区、右侧感知面板、顶部标签页（对话/任务/设置）、底部输入与控制条。按序加载 css + 三个 js。 |
| `css/styles.css` | 全部样式：元宝风格清爽 UI、卡片、感知面板、任务面板、设置页、动画。 |
| `js/engine.js` | 纯逻辑：`classifyIntent` / `intentTrajectory` / `topicContinuity` / `detectDeadline` / `deriveFactors` / `computeTriggerScore` / `decideTier` / `analyze`。双模式导出。 |
| `js/scenarios.js` | 场景数据：主线 `MAIN` 回合脚本、静态场景 `STATIC_A`/`STATIC_B`、主动卡片定义、任务面板项、彩蛋文案。双模式导出。 |
| `js/ui.js` | DOM 渲染 + 事件：聊天渲染、感知面板（可交互）、主动卡片（含"为什么"）、任务面板、设置页、时间快进、反馈微动效、bootstrap。 |
| `tests/_harness.js` | 30 行微型测试运行器（`test()` + 计数 + `ALL PASS`/退出码）。 |
| `tests/engine.test.js` | `engine.js` 全部纯函数单测。 |
| `tests/scenarios.test.js` | 场景数据完整性 + 主线 `analyze` 在关键回合产出正确 tier 的集成测。 |
| `README.md` | 运行说明、Demo 演示脚本、设计要点指引（更新现有文件）。 |

---

## Task 1: 项目骨架与测试运行器

**Files:**
- Create: `tests/_harness.js`
- Create: `tests/engine.test.js`
- Create: `js/engine.js`
- Create: `index.html`
- Create: `css/styles.css`

**Interfaces:**
- Produces: 微型测试运行器 `test(name, fn)` 与 `done()`（挂在 `global`）；`js/engine.js` 暴露空的 `WUCO.engine = {}` 双模式对象供后续任务填充。

- [ ] **Step 1: 写测试运行器**

Create `tests/_harness.js`:

```js
'use strict';
// 零依赖微型测试运行器。用法：require 后调用全局 test(name, fn)，文件末尾调用 done()。
var _passed = 0, _failed = 0;
global.test = function (name, fn) {
  try { fn(); _passed++; console.log('  ✓ ' + name); }
  catch (e) { _failed++; console.log('  ✗ ' + name + '\n      ' + e.message); }
};
global.done = function () {
  console.log('\n' + (_failed ? 'FAILED: ' + _failed + ' / ' + (_passed + _failed)
                               : 'ALL PASS (' + _passed + ')'));
  process.exit(_failed ? 1 : 0);
};
```

- [ ] **Step 2: 写一个会失败的冒烟测试**

Create `tests/engine.test.js`:

```js
'use strict';
require('./_harness');
var engine = require('../js/engine');

test('engine module loads and is an object', function () {
  if (typeof engine !== 'object' || engine === null) throw new Error('engine not an object');
});

done();
```

- [ ] **Step 3: 运行，确认失败**

Run: `node tests/engine.test.js`
Expected: FAIL —— 报 `Cannot find module '../js/engine'`（文件尚不存在）。

- [ ] **Step 4: 写最小 engine.js 使其通过**

Create `js/engine.js`:

```js
;(function (root) {
  'use strict';

  var WUCOEngine = {};

  // ---- 后续任务在此填充：classifyIntent / analyze 等 ----

  // 双模式导出：Node 下走 module.exports，浏览器下挂 window.WUCO.engine
  if (typeof module !== 'undefined' && module.exports) {
    module.exports = WUCOEngine;
  } else {
    root.WUCO = root.WUCO || {};
    root.WUCO.engine = WUCOEngine;
  }
})(typeof window !== 'undefined' ? window : this);
```

- [ ] **Step 5: 运行，确认通过**

Run: `node tests/engine.test.js`
Expected: PASS —— 末行 `ALL PASS (1)`。

- [ ] **Step 6: 写 index.html 骨架与基础样式**

Create `index.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WUCO Proactive Agent · 元宝主动式服务 Demo</title>
  <link rel="stylesheet" href="css/styles.css" />
</head>
<body>
  <header class="topbar">
    <div class="brand">WUCO Proactive Agent <span class="sub">· 会话内实时意图感知</span></div>
    <nav class="tabs">
      <button class="tab active" data-view="chat">对话</button>
      <button class="tab" data-view="tasks">任务面板</button>
      <button class="tab" data-view="settings">主动服务设置</button>
    </nav>
  </header>

  <main class="layout">
    <section class="left">
      <div id="view-chat" class="view active">
        <div id="chat-log" class="chat-log"></div>
        <div id="card-slot" class="card-slot"></div>
        <div class="composer">
          <button id="btn-upload" class="ghost" title="上传文档">📎</button>
          <input id="composer-input" type="text" placeholder="给元宝发消息…" />
          <button id="btn-send" class="primary">发送</button>
        </div>
        <div class="demo-bar">
          <span class="demo-bar-label">演示控制</span>
          <button id="btn-next" class="chip">▶ 下一步</button>
          <button id="btn-fast" class="chip">⏩ 时间快进</button>
          <select id="sel-scenario" class="chip">
            <option value="MAIN">主线：方案推进</option>
            <option value="STATIC_A">场景A：文档待办</option>
            <option value="STATIC_B">场景B：风格复用</option>
          </select>
          <button id="btn-reset" class="chip">↺ 重置</button>
        </div>
      </div>

      <div id="view-tasks" class="view">
        <h2>任务面板</h2>
        <p class="muted">元宝从你的对话与文档中自动整理出的任务。</p>
        <ul id="task-list" class="task-list"></ul>
      </div>

      <div id="view-settings" class="view">
        <h2>主动服务设置</h2>
        <div id="settings-body"></div>
      </div>
    </section>

    <aside class="right">
      <div id="perception" class="perception"></div>
    </aside>
  </main>

  <script src="js/engine.js"></script>
  <script src="js/scenarios.js"></script>
  <script src="js/ui.js"></script>
</body>
</html>
```

Create `css/styles.css` (基础布局，后续任务追加组件样式):

```css
:root{
  --bg:#f5f7fa; --panel:#ffffff; --ink:#1f2329; --muted:#8a94a6;
  --brand:#2b6cff; --brand-soft:#eaf1ff; --line:#e6e9ef;
  --ok:#1fb96a; --warn:#ff8a00; --urgent:#f5455c; --radius:14px;
  font-family:-apple-system,"Segoe UI","PingFang SC","Microsoft YaHei",sans-serif;
}
*{box-sizing:border-box;} html,body{margin:0;height:100%;background:var(--bg);color:var(--ink);}
.topbar{display:flex;align-items:center;justify-content:space-between;padding:12px 20px;background:var(--panel);border-bottom:1px solid var(--line);}
.brand{font-weight:700;font-size:16px;} .brand .sub{color:var(--muted);font-weight:400;font-size:13px;}
.tabs{display:flex;gap:6px;}
.tab{border:none;background:transparent;padding:8px 14px;border-radius:10px;cursor:pointer;color:var(--muted);font-size:14px;}
.tab.active{background:var(--brand-soft);color:var(--brand);font-weight:600;}
.layout{display:flex;gap:16px;padding:16px;height:calc(100% - 57px);}
.left{flex:1;min-width:0;background:var(--panel);border-radius:var(--radius);border:1px solid var(--line);display:flex;flex-direction:column;overflow:hidden;}
.right{width:340px;flex:none;}
.view{display:none;flex:1;flex-direction:column;min-height:0;padding:16px;overflow:auto;}
.view.active{display:flex;}
.chat-log{flex:1;overflow:auto;display:flex;flex-direction:column;gap:12px;padding-right:6px;}
.composer{display:flex;gap:8px;padding-top:12px;border-top:1px solid var(--line);margin-top:12px;}
.composer input{flex:1;padding:10px 12px;border:1px solid var(--line);border-radius:10px;font-size:14px;}
.primary{background:var(--brand);color:#fff;border:none;border-radius:10px;padding:10px 16px;cursor:pointer;}
.ghost{background:transparent;border:1px solid var(--line);border-radius:10px;padding:8px 10px;cursor:pointer;}
.demo-bar{display:flex;align-items:center;gap:8px;margin-top:10px;flex-wrap:wrap;}
.demo-bar-label{font-size:12px;color:var(--muted);} 
.chip{font-size:13px;border:1px solid var(--line);background:#fff;border-radius:999px;padding:6px 12px;cursor:pointer;}
.muted{color:var(--muted);} h2{font-size:18px;margin:0 0 8px;}
```

- [ ] **Step 7: 浏览器手动验证骨架**

双击 `index.html`（或在浏览器打开 `file://.../index.html`）。
Expected: 看到顶栏「WUCO Proactive Agent」、三个标签、左侧空聊天区与输入框、底部演示控制条、右侧空白面板。控制台无报错。（按钮暂无功能属正常。）

- [ ] **Step 8: 提交**

```bash
git add index.html css/styles.css js/engine.js tests/_harness.js tests/engine.test.js
git commit -m "chore: scaffold demo shell and zero-dep test runner"
```

---

## Task 2: 意图分类与爬升轨迹（engine）

**Files:**
- Modify: `js/engine.js`
- Modify: `tests/engine.test.js`

**Interfaces:**
- Produces:
  - `WUCO.engine.STAGE = { NONE:'none', COGNITION:'cognition', METHOD:'method', ACTION:'action' }`
  - `classifyIntent(text) -> { stage: string, matched: string|null }`
  - `intentTrajectory(stages: string[]) -> { ascending: boolean, peak: string, span: number }`

- [ ] **Step 1: 写失败测试**

Append to `tests/engine.test.js`（在 `done();` 之前）:

```js
test('classifyIntent: 行动阶段优先识别截止/提交', function () {
  if (engine.classifyIntent('提交截止时间是什么').stage !== engine.STAGE.ACTION) throw new Error('应为 action');
});
test('classifyIntent: 方法阶段识别怎么/框架', function () {
  if (engine.classifyIntent('这个方案怎么搭框架').stage !== engine.STAGE.METHOD) throw new Error('应为 method');
});
test('classifyIntent: 认知阶段识别是什么/有哪些', function () {
  if (engine.classifyIntent('主动式AI是什么').stage !== engine.STAGE.COGNITION) throw new Error('应为 cognition');
});
test('classifyIntent: 无匹配返回 none', function () {
  if (engine.classifyIntent('今天天气不错').stage !== engine.STAGE.NONE) throw new Error('应为 none');
});
test('intentTrajectory: 认知→方法→行动判为爬升', function () {
  var t = engine.intentTrajectory(['cognition','method','action']);
  if (!t.ascending || t.peak !== 'action') throw new Error('应爬升且 peak=action');
});
test('intentTrajectory: 回落不算爬升', function () {
  if (engine.intentTrajectory(['action','cognition']).ascending) throw new Error('不应爬升');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node tests/engine.test.js`
Expected: FAIL —— `engine.classifyIntent is not a function`。

- [ ] **Step 3: 实现**

在 `js/engine.js` 的 `var WUCOEngine = {};` 之后、导出之前插入:

```js
  var STAGE = { NONE:'none', COGNITION:'cognition', METHOD:'method', ACTION:'action' };
  var RANK  = { none:0, cognition:1, method:2, action:3 };

  // 注意：触发只看"意图语义升级"，不看次数、不看 idle/时间 —— 见设计原则。
  var PATTERNS = {
    action:    [/截止|deadline|期限|due|什么时候|提交|交付|交给|发给|帮我写|帮我生成|帮我做出来|落地/],
    method:    [/怎么|如何|怎样|步骤|框架|大纲|规划|流程|方案|搭建|设计|制定/],
    cognition: [/是什么|什么是|有哪些|哪些|介绍|了解|区别|为什么|原理|概念|定义/]
  };

  function classifyIntent(text) {
    if (!text) return { stage: STAGE.NONE, matched: null };
    var order = ['action', 'method', 'cognition']; // 高阶段优先
    for (var i = 0; i < order.length; i++) {
      var pats = PATTERNS[order[i]];
      for (var j = 0; j < pats.length; j++) {
        var m = text.match(pats[j]);
        if (m) return { stage: STAGE[order[i].toUpperCase()], matched: m[0] };
      }
    }
    return { stage: STAGE.NONE, matched: null };
  }

  function rankToStage(r) { return ['none','cognition','method','action'][r] || STAGE.NONE; }

  function intentTrajectory(stages) {
    var ranks = (stages || []).map(function (s) { return RANK[s] || 0; })
                              .filter(function (r) { return r > 0; });
    if (!ranks.length) return { ascending:false, peak:STAGE.NONE, span:0 };
    var ascending = ranks.length >= 2;
    for (var i = 1; i < ranks.length; i++) if (ranks[i] < ranks[i-1]) { ascending = false; break; }
    return { ascending: ascending, peak: rankToStage(Math.max.apply(null, ranks)), span: ranks.length };
  }

  WUCOEngine.STAGE = STAGE;
  WUCOEngine.classifyIntent = classifyIntent;
  WUCOEngine.intentTrajectory = intentTrajectory;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node tests/engine.test.js`
Expected: PASS —— `ALL PASS (7)`。

- [ ] **Step 5: 提交**

```bash
git add js/engine.js tests/engine.test.js
git commit -m "feat: intent classification and ascending-trajectory detection"
```

---

## Task 3: 信号提取——主题连续性与截止检测（engine）

**Files:**
- Modify: `js/engine.js`
- Modify: `tests/engine.test.js`

**Interfaces:**
- Produces:
  - `topicContinuity(messages: string[]) -> number`（0..1，基于最近 4 条用户消息的 CJK 二元组 + 英文词重叠）
  - `detectDeadline(text) -> { found: boolean, label?: string }`

- [ ] **Step 1: 写失败测试**

Append to `tests/engine.test.js`（`done();` 之前）:

```js
test('topicContinuity: 同主题连续提问得分高', function () {
  var c = engine.topicContinuity(['主动式AI是什么','主动式服务方案怎么做','主动服务的框架']);
  if (!(c > 0.2)) throw new Error('同主题应 > 0.2，实际 ' + c);
});
test('topicContinuity: 无关话题得分低', function () {
  var c = engine.topicContinuity(['今天天气怎么样','晚饭吃什么']);
  if (c > 0.2) throw new Error('无关应 <= 0.2，实际 ' + c);
});
test('topicContinuity: 少于两条返回 0', function () {
  if (engine.topicContinuity(['只有一条']) !== 0) throw new Error('应为 0');
});
test('detectDeadline: 识别 2026-07-19 形式', function () {
  if (!engine.detectDeadline('作品提交截止时间是 2026-07-19 23:59:59').found) throw new Error('应识别日期');
});
test('detectDeadline: 识别中文年月日', function () {
  if (!engine.detectDeadline('截止到2026年7月19日').found) throw new Error('应识别中文日期');
});
test('detectDeadline: 无截止返回 false', function () {
  if (engine.detectDeadline('帮我写个开头').found) throw new Error('不应识别');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node tests/engine.test.js`
Expected: FAIL —— `engine.topicContinuity is not a function`。

- [ ] **Step 3: 实现**

在 `js/engine.js` 的 `intentTrajectory` 实现之后插入:

```js
  function tokenize(text) {
    if (!text) return [];
    var tokens = [];
    var aw = text.match(/[a-zA-Z]{2,}/g) || [];
    for (var k = 0; k < aw.length; k++) tokens.push(aw[k].toLowerCase());
    var cjk = text.match(/[一-龥]/g) || []; // 用二元组捕捉子串重叠
    for (var i = 0; i < cjk.length - 1; i++) tokens.push(cjk[i] + cjk[i+1]);
    return tokens;
  }

  function topicContinuity(messages) {
    var recent = (messages || []).slice(-4);
    if (recent.length < 2) return 0;
    var sets = recent.map(function (t) {
      var s = {}; tokenize(t).forEach(function (tok) { s[tok] = true; }); return s;
    });
    var total = 0, pairs = 0;
    for (var i = 1; i < sets.length; i++) {
      var a = sets[i-1], b = sets[i], inter = 0, bn = 0, an = 0, key;
      for (key in b) if (b.hasOwnProperty(key)) { bn++; if (a[key]) inter++; }
      for (key in a) if (a.hasOwnProperty(key)) an++;
      var denom = Math.min(an, bn) || 1;
      total += inter / denom; pairs++;
    }
    return pairs ? Math.min(1, total / pairs) : 0;
  }

  function detectDeadline(text) {
    if (!text) return { found:false };
    var iso = text.match(/(\d{4})[-/年](\d{1,2})[-/月](\d{1,2})/);
    if (iso) return { found:true, label:iso[0] };
    if (/截止|deadline|期限|due|提交时间|交付时间/.test(text)) return { found:true, label:'检测到截止节点' };
    return { found:false };
  }

  WUCOEngine.topicContinuity = topicContinuity;
  WUCOEngine.detectDeadline = detectDeadline;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node tests/engine.test.js`
Expected: PASS —— `ALL PASS (13)`。

- [ ] **Step 5: 提交**

```bash
git add js/engine.js tests/engine.test.js
git commit -m "feat: topic-continuity and deadline signal extraction"
```

---

## Task 4: 触发评分、分级与编排器 analyze（engine）

**Files:**
- Modify: `js/engine.js`
- Modify: `tests/engine.test.js`

**Interfaces:**
- Produces:
  - `deriveFactors(signals) -> { demand, timeliness, confidence, acceptance, interruptionCost, taskSwitchPenalty }`（各 0..1）
  - `computeTriggerScore(factors) -> number`（整数）
  - `decideTier(score, ctx) -> 'silent'|'ambient'|'card'|'urgent'`
  - `analyze(userMessages: string[], ctx) -> { stages, trajectory, signals, factors, score, tier }`
  - signals 字段：`{ peakStage, ascending, continuity, deadline, userTyping, recentlyRejected, acceptanceHistory, wouldSwitchTask }`
  - ctx 字段：`{ proactivityLevel:'off'|'low'|'standard'|'active', userTyping, recentlyRejected, acceptanceHistory, wouldSwitchTask, deadlineRisk }`

- [ ] **Step 1: 写失败测试**

Append to `tests/engine.test.js`（`done();` 之前）:

```js
test('computeTriggerScore: 公式正确（满需求无打扰）', function () {
  var s = engine.computeTriggerScore({demand:1,timeliness:1,confidence:1,acceptance:1,interruptionCost:0,taskSwitchPenalty:0});
  if (s !== 100) throw new Error('应为 100，实际 ' + s);
});
test('computeTriggerScore: 自发任务切换重罚', function () {
  var base = engine.computeTriggerScore({demand:1,timeliness:1,confidence:1,acceptance:1,interruptionCost:0,taskSwitchPenalty:0});
  var pen  = engine.computeTriggerScore({demand:1,timeliness:1,confidence:1,acceptance:1,interruptionCost:0,taskSwitchPenalty:1});
  if (!(pen === base - 50)) throw new Error('λ=0.5 应扣 50，实际差 ' + (base - pen));
});
test('analyze: 单条认知问句 → silent', function () {
  var r = engine.analyze(['主动式AI是什么'], {proactivityLevel:'standard'});
  if (r.tier !== 'silent') throw new Error('应 silent，实际 ' + r.tier + ' score=' + r.score);
});
test('analyze: 认知→方法→行动+截止 → card', function () {
  var r = engine.analyze(
    ['主动式AI是什么','主动服务方案怎么搭框架','提交截止时间是 2026-07-19'],
    {proactivityLevel:'standard'});
  if (r.tier !== 'card') throw new Error('应 card，实际 ' + r.tier + ' score=' + r.score);
});
test('analyze: 主动程度=off 时永不触发', function () {
  var r = engine.analyze(
    ['主动式AI是什么','主动服务方案怎么搭框架','提交截止时间是 2026-07-19'],
    {proactivityLevel:'off'});
  if (r.tier !== 'silent') throw new Error('off 应 silent，实际 ' + r.tier);
});
test('analyze: 刚被拒绝则打扰成本上升，压制触发', function () {
  var withRej = engine.analyze(
    ['主动式AI是什么','主动服务方案怎么搭框架','提交截止时间是 2026-07-19'],
    {proactivityLevel:'standard', recentlyRejected:true});
  if (withRej.tier === 'card') throw new Error('刚拒绝后不应立刻再弹 card');
});
```

- [ ] **Step 2: 运行，确认失败**

Run: `node tests/engine.test.js`
Expected: FAIL —— `engine.computeTriggerScore is not a function`。

- [ ] **Step 3: 实现**

在 `js/engine.js` 的 `detectDeadline` 之后插入:

```js
  function clamp01(x) { return Math.max(0, Math.min(1, x)); }

  var STAGE_SCORE = { none:0.1, cognition:0.3, method:0.55, action:0.9 };

  function deriveFactors(s) {
    s = s || {};
    var demand = clamp01((STAGE_SCORE[s.peakStage] || 0.1)
                         + (s.ascending ? 0.05 : 0)
                         + 0.3 * (s.continuity || 0));
    var timeliness = s.deadline ? 1.0 : 0.5;
    var confidence = clamp01(0.5 + 0.4 * (s.continuity || 0) + (s.peakStage === 'action' ? 0.15 : 0));
    var acceptance = clamp01(typeof s.acceptanceHistory === 'number' ? s.acceptanceHistory : 0.6);
    var interruptionCost = clamp01((s.userTyping ? 0.5 : 0.1) + (s.recentlyRejected ? 0.5 : 0));
    var taskSwitchPenalty = s.wouldSwitchTask ? 1 : 0;
    return { demand:demand, timeliness:timeliness, confidence:confidence,
             acceptance:acceptance, interruptionCost:interruptionCost, taskSwitchPenalty:taskSwitchPenalty };
  }

  var LAMBDA = 0.5;
  function computeTriggerScore(f) {
    var value = clamp01(f.demand) * clamp01(f.timeliness) * clamp01(f.confidence) * clamp01(f.acceptance);
    var score = value * 100 - clamp01(f.interruptionCost) * 40 - LAMBDA * clamp01(f.taskSwitchPenalty || 0) * 100;
    return Math.round(score);
  }

  var THRESHOLDS = { off: Infinity, low: 50, standard: 38, active: 28 };
  function decideTier(score, ctx) {
    ctx = ctx || {};
    var th = THRESHOLDS[ctx.proactivityLevel || 'standard'];
    if (ctx.deadlineRisk && score >= th - 12) return 'urgent';
    if (score < th) return 'silent';
    if (score < th + 8) return 'ambient';
    return 'card';
  }

  function analyze(userMessages, ctx) {
    ctx = ctx || {};
    var msgs = (userMessages || []).filter(function (m) { return !!m; });
    var stages = msgs.map(function (m) { return classifyIntent(m).stage; });
    var traj = intentTrajectory(stages);
    var continuity = topicContinuity(msgs);
    var anyDeadline = msgs.some(function (m) { return detectDeadline(m).found; });
    var signals = {
      peakStage: traj.peak, ascending: traj.ascending, continuity: continuity,
      deadline: anyDeadline, userTyping: !!ctx.userTyping, recentlyRejected: !!ctx.recentlyRejected,
      acceptanceHistory: ctx.acceptanceHistory, wouldSwitchTask: !!ctx.wouldSwitchTask
    };
    var factors = deriveFactors(signals);
    var score = computeTriggerScore(factors);
    var tier = decideTier(score, { proactivityLevel: ctx.proactivityLevel,
                                    deadlineRisk: ctx.deadlineRisk || anyDeadline && traj.peak === 'action' && false });
    return { stages:stages, trajectory:traj, signals:signals, factors:factors, score:score, tier:tier };
  }

  WUCOEngine.deriveFactors = deriveFactors;
  WUCOEngine.computeTriggerScore = computeTriggerScore;
  WUCOEngine.decideTier = decideTier;
  WUCOEngine.analyze = analyze;
```

- [ ] **Step 4: 运行，确认通过**

Run: `node tests/engine.test.js`
Expected: PASS —— `ALL PASS (19)`。若 `analyze: 认知→方法→行动+截止 → card` 失败，调高 `STAGE_SCORE.action` 或调低 `THRESHOLDS.standard` 后重跑，直至该回合恰为 `card`、单条认知为 `silent`。

- [ ] **Step 5: 提交**

```bash
git add js/engine.js tests/engine.test.js
git commit -m "feat: trigger scoring, tiering and analyze orchestrator"
```

---

## Task 5: 场景数据与状态机（scenarios）

**Files:**
- Create: `js/scenarios.js`
- Create: `tests/scenarios.test.js`

**Interfaces:**
- Consumes: `WUCO.engine.analyze`（Node 下 `require('../js/engine')`）。
- Produces: `WUCO.scenarios`，含：
  - `MAIN`, `STATIC_A`, `STATIC_B`：`{ id, title, turns:[{user, ai, upload?, deadlineDays?}], card:{...}, tasks:[{label,status}], easterEgg?:string }`
  - 卡片定义 `card`：`{ title, body, why:[string], actions:[{key,label,kind}] }`
  - `get(id) -> scenario`

- [ ] **Step 1: 写失败测试**

Create `tests/scenarios.test.js`:

```js
'use strict';
require('./_harness');
var engine = require('../js/engine');
var S = require('../js/scenarios');

test('MAIN 场景结构完整', function () {
  var m = S.get('MAIN');
  if (!m || !m.turns.length) throw new Error('MAIN 应有 turns');
  if (!m.card || !m.card.why.length) throw new Error('MAIN.card 应含 why 解释');
  if (!m.tasks.length) throw new Error('MAIN 应有 tasks');
  if (!m.easterEgg) throw new Error('MAIN 应有彩蛋文案');
});

test('MAIN 逐回合累积后在最后一回合产出 card', function () {
  var m = S.get('MAIN'), history = [];
  var tiers = m.turns.map(function (t) {
    history.push(t.user);
    return engine.analyze(history, { proactivityLevel:'standard' }).tier;
  });
  if (tiers[0] !== 'silent') throw new Error('首回合应 silent，实际 ' + tiers[0]);
  if (tiers[tiers.length - 1] !== 'card') throw new Error('末回合应 card，实际 ' + tiers[tiers.length-1]);
});

test('三个场景都能 get 到', function () {
  ['MAIN','STATIC_A','STATIC_B'].forEach(function (id) {
    if (!S.get(id)) throw new Error('缺场景 ' + id);
  });
});

done();
```

- [ ] **Step 2: 运行，确认失败**

Run: `node tests/scenarios.test.js`
Expected: FAIL —— `Cannot find module '../js/scenarios'`。

- [ ] **Step 3: 实现**

Create `js/scenarios.js`:

```js
;(function (root) {
  'use strict';

  var MAIN = {
    id: 'MAIN',
    title: '主线：方案推进',
    turns: [
      { user: '主动式 AI 是什么？有哪些做法？',
        ai: '主动式 AI 指无需用户显式提问、由系统主动感知需求并触达服务。常见做法有：定时简报（如 ChatGPT Pulse 隔夜生成）、每日聚合简报（如 Anthropic Orbit）、以及指令驱动的 agent（如 Gemini）。' },
      { user: '这种主动服务的方案一般怎么搭框架？',
        ai: '通常分五层：上下文感知 → 需求预判 → 触达判断 → 主动触达 → 反馈学习。关键在触达判断要用评分模型权衡价值与打扰成本。' },
      { user: '我在准备一个参赛方案，提交截止时间是 2026-07-19，帮我看看怎么推进。',
        ai: '收到。我注意到你已围绕「主动式 AI 方案」连续深入，并出现了明确截止节点。',
        upload: '活动说明.pdf', deadlineDays: 20 }
    ],
    card: {
      title: '我发现你可能正在准备一个复杂任务',
      body: '你围绕「主动式 AI 方案」的提问从“是什么”一路深入到“截止时间/怎么推进”，并上传了说明文档。要不要我把当前信息整理成一份方案框架？',
      why: [
        '意图轨迹：认知 → 方法 → 行动（持续爬升）',
        '主题连续性高：三轮均围绕“主动式 AI 方案”',
        '检测到截止节点：2026-07-19',
        '当前非高频输入，打扰成本低'
      ],
      actions: [
        { key:'generate', label:'生成方案框架', kind:'primary' },
        { key:'later',    label:'稍后提醒',     kind:'ghost' },
        { key:'mute',     label:'不再提示此类', kind:'ghost' }
      ]
    },
    tasks: [
      { label:'明确作品命名与定位', status:'done' },
      { label:'设计主动触发机制',   status:'doing' },
      { label:'制作 Demo 主流程',   status:'todo' },
      { label:'准备提交材料',       status:'todo' }
    ],
    easterEgg: '顺便说——你正在看的这套参赛方案本身，就是我这样一步步帮你推进出来的。'
  };

  var STATIC_A = {
    id: 'STATIC_A',
    title: '场景A：文档待办',
    turns: [
      { user: '帮我总结一下这份活动说明文档。', ai: '已总结：这是一份黑客松赛题说明，含背景、方向、作品形式与提交要求。',
        upload:'活动说明.pdf' }
    ],
    card: {
      title: '这份文档不只是要总结，还藏着要做的事',
      body: '我在文档里发现 4 个关键待办和 1 个截止时间。要不要生成提交清单并排出时间规划？',
      why: ['文档含“提交要求/评审标准/截止时间”等行动性内容', '检测到截止节点：2026-07-19'],
      actions: [
        { key:'generate', label:'生成提交清单', kind:'primary' },
        { key:'later',    label:'稍后',         kind:'ghost' },
        { key:'mute',     label:'不再提示此类', kind:'ghost' }
      ]
    },
    tasks: [
      { label:'确认作品命名', status:'todo' },
      { label:'完成 Demo 主流程', status:'todo' },
      { label:'撰写评审说明页', status:'todo' },
      { label:'7-19 前提交', status:'todo' }
    ]
  };

  var STATIC_B = {
    id: 'STATIC_B',
    title: '场景B：风格复用',
    turns: [
      { user: '把这段再口语化润色一下，简短点。', ai: '已按更口语、更精简的风格改写完成。' }
    ],
    card: {
      title: '我注意到你稳定的写作偏好',
      body: '近几次改写你都偏好“口语化、精简、逻辑清晰”。要不要把这次也自动套用该风格，以后默认沿用？',
      why: ['多次改写均指向同一风格偏好', '复用偏好可减少重复指令'],
      actions: [
        { key:'generate', label:'套用并记住该风格', kind:'primary' },
        { key:'later',    label:'仅这次',           kind:'ghost' },
        { key:'mute',     label:'不再提示此类',     kind:'ghost' }
      ]
    },
    tasks: [
      { label:'记住写作风格偏好', status:'done' },
      { label:'下次自动套用', status:'doing' }
    ]
  };

  var ALL = { MAIN:MAIN, STATIC_A:STATIC_A, STATIC_B:STATIC_B };
  var SCEN = { get: function (id) { return ALL[id] || null; }, MAIN:MAIN, STATIC_A:STATIC_A, STATIC_B:STATIC_B };

  if (typeof module !== 'undefined' && module.exports) module.exports = SCEN;
  else { root.WUCO = root.WUCO || {}; root.WUCO.scenarios = SCEN; }
})(typeof window !== 'undefined' ? window : this);
```

- [ ] **Step 4: 运行，确认通过**

Run: `node tests/scenarios.test.js`
Expected: PASS —— `ALL PASS (3)`。若「末回合应 card」失败，检查第 3 回合 user 文案是否同时含行动词（“截止/帮我”）与主题词（“主动式/方案”）以拉高 demand 与 continuity。

- [ ] **Step 5: 提交**

```bash
git add js/scenarios.js tests/scenarios.test.js
git commit -m "feat: scenario data and state-machine with main/static scenarios"
```

---

## Task 6: 聊天渲染与对话推进（ui）

**Files:**
- Create: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `WUCO.scenarios.get`, `WUCO.engine.analyze`。
- Produces: 浏览器全局 `WUCO.app`，含 `state`（`{ scenarioId, turnIndex, history:[], proactivityLevel, recentlyRejected, accepted:false }`）、`renderChat()`、`advance()`、`resetApp()`、`pushMessage(role, text)`。

- [ ] **Step 1: 写 ui.js bootstrap 与聊天渲染**

Create `js/ui.js`:

```js
;(function () {
  'use strict';
  var engine = window.WUCO.engine, scenarios = window.WUCO.scenarios;

  var state = {
    scenarioId: 'MAIN', turnIndex: 0, history: [],
    proactivityLevel: 'standard', recentlyRejected: false, accepted: false,
    messages: [] // {role:'user'|'ai', text}
  };

  var $ = function (sel) { return document.querySelector(sel); };
  var el = function (tag, cls, html) {
    var n = document.createElement(tag);
    if (cls) n.className = cls;
    if (html != null) n.innerHTML = html;
    return n;
  };

  function pushMessage(role, text) {
    state.messages.push({ role: role, text: text });
    if (role === 'user') state.history.push(text);
  }

  function renderChat() {
    var log = $('#chat-log'); log.innerHTML = '';
    state.messages.forEach(function (m) {
      var row = el('div', 'msg msg-' + m.role);
      row.appendChild(el('div', 'bubble', m.text));
      log.appendChild(row);
    });
    log.scrollTop = log.scrollHeight;
  }

  function currentScenario() { return scenarios.get(state.scenarioId); }

  function advance() {
    var sc = currentScenario();
    if (state.turnIndex >= sc.turns.length) return;
    var turn = sc.turns[state.turnIndex];
    if (turn.upload) pushMessage('user', '📎 已上传《' + turn.upload + '》');
    pushMessage('user', turn.user);
    pushMessage('ai', turn.ai);
    state.turnIndex++;
    renderChat();
    refreshPerception();          // Task 7 实现
    maybeFireCard();              // Task 8 实现
  }

  function resetApp() {
    state.turnIndex = 0; state.history = []; state.messages = [];
    state.recentlyRejected = false; state.accepted = false;
    renderChat(); refreshPerception(); clearCard(); renderTasks();
  }

  // 占位，后续任务实现；先定义空函数避免报错
  function refreshPerception() {}
  function maybeFireCard() {}
  function clearCard() { var s = $('#card-slot'); if (s) s.innerHTML = ''; }
  function renderTasks() {}

  function bindUI() {
    $('#btn-next').addEventListener('click', advance);
    $('#btn-reset').addEventListener('click', resetApp);
    $('#sel-scenario').addEventListener('change', function (e) {
      state.scenarioId = e.target.value; resetApp();
    });
    $('#btn-send').addEventListener('click', function () {
      var inp = $('#composer-input'); var v = (inp.value || '').trim();
      if (!v) { advance(); return; } // 空输入等同“下一步”，方便演示
      pushMessage('user', v); inp.value = '';
      pushMessage('ai', '（演示模式）已收到：' + v);
      renderChat(); refreshPerception(); maybeFireCard();
    });
    Array.prototype.forEach.call(document.querySelectorAll('.tab'), function (t) {
      t.addEventListener('click', function () {
        document.querySelectorAll('.tab').forEach(function (x){ x.classList.remove('active'); });
        document.querySelectorAll('.view').forEach(function (x){ x.classList.remove('active'); });
        t.classList.add('active');
        $('#view-' + t.dataset.view).classList.add('active');
      });
    });
  }

  document.addEventListener('DOMContentLoaded', function () {
    bindUI(); renderChat(); refreshPerception(); renderTasks();
  });

  window.WUCO.app = {
    state: state, renderChat: renderChat, advance: advance,
    resetApp: resetApp, pushMessage: pushMessage,
    _setImpl: function (impl) { // 供后续任务替换占位实现
      if (impl.refreshPerception) refreshPerception = impl.refreshPerception;
      if (impl.maybeFireCard) maybeFireCard = impl.maybeFireCard;
      if (impl.renderTasks) renderTasks = impl.renderTasks;
    }
  };
})();
```

> 说明：占位函数 `refreshPerception`/`maybeFireCard`/`renderTasks` 将在 Task 7/8/9 内**直接在本文件改写**（不是靠 `_setImpl`；`_setImpl` 仅为防御性留口，可不使用）。后续任务在 `js/ui.js` 内就地替换这三个空实现。

Append to `css/styles.css`:

```css
.msg{display:flex;} .msg-user{justify-content:flex-end;}
.bubble{max-width:78%;padding:10px 14px;border-radius:14px;font-size:14px;line-height:1.55;white-space:pre-wrap;}
.msg-ai .bubble{background:#f1f3f7;border-top-left-radius:4px;}
.msg-user .bubble{background:var(--brand);color:#fff;border-top-right-radius:4px;}
```

- [ ] **Step 2: 浏览器手动验证**

双击 `index.html`，反复点击「▶ 下一步」。
Expected: 主线三回合依次出现——先“已上传”气泡、用户问句、元宝回答；最后一回合含上传文档气泡。切换场景下拉框会清空重来。点「↺ 重置」回到空。控制台无报错。

- [ ] **Step 3: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: chat rendering and scripted conversation advance"
```

---

## Task 7: 可交互感知面板（ui）

**Files:**
- Modify: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `engine.analyze`, `state`。
- Produces: 就地替换 `refreshPerception()`，实时渲染信号 + 触发分 + 主动程度滑块（可交互，改变即重算）。

- [ ] **Step 1: 实现 refreshPerception 与交互**

在 `js/ui.js` 中，将占位的 `function refreshPerception() {}` 整体替换为:

```js
  function ctxFromState() {
    return { proactivityLevel: state.proactivityLevel, recentlyRejected: state.recentlyRejected,
             acceptanceHistory: state.accepted ? 0.9 : 0.6 };
  }

  function tierLabel(t) {
    return { silent:'静默', ambient:'轻提示', card:'主动卡片', urgent:'紧急提醒' }[t] || t;
  }

  function refreshPerception() {
    var p = $('#perception'); if (!p) return;
    var r = engine.analyze(state.history, ctxFromState());
    var f = r.factors;
    var bar = function (label, v) {
      var pct = Math.round(v * 100);
      return '<div class="pf-row"><span>' + label + '</span>'
           + '<div class="pf-track"><div class="pf-fill" style="width:' + pct + '%"></div></div>'
           + '<b>' + pct + '</b></div>';
    };
    var stageZh = { none:'—', cognition:'认知', method:'方法', action:'行动' };
    p.innerHTML =
      '<div class="pf-head">感知面板 <span class="muted">实时</span></div>'
      + '<div class="pf-signal">意图阶段：<b>' + stageZh[r.signals.peakStage] + '</b>'
      + (r.signals.ascending ? ' <span class="tag tag-ok">↑爬升中</span>' : '') + '</div>'
      + '<div class="pf-signal">截止节点：<b>' + (r.signals.deadline ? '已检测' : '无') + '</b></div>'
      + bar('需求强度', f.demand) + bar('时效性', f.timeliness)
      + bar('置信度', f.confidence) + bar('接受度', f.acceptance)
      + bar('打扰成本', f.interruptionCost)
      + '<div class="pf-score">触发分 <b>' + r.score + '</b> → '
      + '<span class="tag tag-' + r.tier + '">' + tierLabel(r.tier) + '</span></div>'
      + '<div class="pf-ctl"><label>主动程度</label>'
      + '<input id="pf-level" type="range" min="0" max="3" step="1" value="' + levelToNum(state.proactivityLevel) + '"></div>'
      + '<div class="pf-ctl-val muted">' + levelZh(state.proactivityLevel) + '（拖动可当场调节，触达即时变化）</div>';
    var slider = $('#pf-level');
    if (slider) slider.addEventListener('input', function (e) {
      state.proactivityLevel = numToLevel(+e.target.value);
      refreshPerception(); maybeFireCard();
    });
  }

  function levelToNum(l){ return {off:0,low:1,standard:2,active:3}[l]; }
  function numToLevel(n){ return ['off','low','standard','active'][n]; }
  function levelZh(l){ return {off:'关闭',low:'低',standard:'标准',active:'积极'}[l]; }
```

Append to `css/styles.css`:

```css
.perception{background:var(--panel);border:1px solid var(--line);border-radius:var(--radius);padding:16px;height:100%;overflow:auto;}
.pf-head{font-weight:700;margin-bottom:12px;}
.pf-signal{font-size:13px;margin:6px 0;}
.pf-row{display:flex;align-items:center;gap:8px;font-size:12px;margin:8px 0;}
.pf-row span{width:60px;color:var(--muted);} .pf-row b{width:28px;text-align:right;}
.pf-track{flex:1;height:8px;background:#eef1f6;border-radius:99px;overflow:hidden;}
.pf-fill{height:100%;background:var(--brand);transition:width .4s;}
.pf-score{margin-top:14px;padding-top:12px;border-top:1px solid var(--line);font-size:14px;}
.tag{font-size:12px;padding:2px 8px;border-radius:99px;}
.tag-ok{background:#e7f8ef;color:var(--ok);} .tag-silent{background:#eef1f6;color:var(--muted);}
.tag-ambient{background:#fff4e5;color:var(--warn);} .tag-card{background:var(--brand-soft);color:var(--brand);}
.tag-urgent{background:#ffe8eb;color:var(--urgent);}
.pf-ctl{margin-top:14px;display:flex;align-items:center;gap:10px;} .pf-ctl input{flex:1;}
.pf-ctl-val{font-size:12px;margin-top:4px;}
```

- [ ] **Step 2: 浏览器手动验证**

双击 `index.html`，逐步点「▶ 下一步」并观察右侧面板。
Expected: 触发分随回合升高；意图阶段从“认知→方法→行动”，第 3 回合出现“↑爬升中”与“截止已检测”，tier 标签从“静默”变为“主动卡片”。拖动「主动程度」滑块到“关闭”时 tier 立刻变“静默”，到“积极”时更易出现“主动卡片”。

- [ ] **Step 3: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: interactive perception panel with live signals and proactivity slider"
```

---

## Task 8: 主动服务卡片（含"为什么"与反馈微动效）（ui）

**Files:**
- Modify: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `engine.analyze`, `currentScenario`, `state`。
- Produces: 就地替换 `maybeFireCard()`；新增 `renderCard(card)`、`onCardAction(key)`。卡片仅在 `tier==='card'||'urgent'` 且未接受/未静默时出现。

- [ ] **Step 1: 实现卡片逻辑**

在 `js/ui.js` 中，将占位 `function maybeFireCard() {}` 整体替换，并补充辅助函数:

```js
  var cardMuted = false;

  function maybeFireCard() {
    if (cardMuted || state.accepted) { clearCard(); return; }
    var r = engine.analyze(state.history, ctxFromState());
    if (r.tier === 'card' || r.tier === 'urgent') renderCard(currentScenario().card, r);
    else clearCard();
  }

  function renderCard(card, r) {
    var slot = $('#card-slot'); if (!slot) return;
    var whyId = 'why-' + Date.now();
    var actions = card.actions.map(function (a) {
      return '<button class="' + (a.kind === 'primary' ? 'primary' : 'ghost') + '" data-act="' + a.key + '">' + a.label + '</button>';
    }).join('');
    slot.innerHTML =
      '<div class="pcard">'
      + '<div class="pcard-head">🔮 ' + card.title
      + '<button class="why-btn" data-why="' + whyId + '">💡 为什么提示我</button></div>'
      + '<div class="pcard-body">' + card.body + '</div>'
      + '<div class="pcard-why" id="' + whyId + '" hidden><ul>'
      + card.why.map(function (w){ return '<li>' + w + '</li>'; }).join('')
      + '</ul><div class="muted">触发分 ' + r.score + '，超过当前“' + levelZh(state.proactivityLevel) + '”阈值才出现。</div></div>'
      + '<div class="pcard-actions">' + actions + '</div></div>';

    slot.querySelector('.why-btn').addEventListener('click', function () {
      var w = document.getElementById(whyId); w.hidden = !w.hidden;
    });
    Array.prototype.forEach.call(slot.querySelectorAll('[data-act]'), function (b) {
      b.addEventListener('click', function () { onCardAction(b.dataset.act); });
    });
  }

  function onCardAction(key) {
    var sc = currentScenario();
    if (key === 'generate') {
      state.accepted = true; state.recentlyRejected = false;
      pushMessage('ai', buildGeneratedReply(sc));
      renderChat(); renderTasks(); clearCard(); refreshPerception();
      flashAcceptance();
    } else if (key === 'later') {
      state.recentlyRejected = true; clearCard(); refreshPerception();
    } else if (key === 'mute') {
      cardMuted = true; state.recentlyRejected = true; clearCard(); refreshPerception();
    }
  }

  function buildGeneratedReply(sc) {
    if (sc.id === 'MAIN')
      return '已为你整理「方案框架」：\n① 项目定位　② 核心场景　③ 主动触发机制\n④ Demo 脚本　⑤ 提交前待办\n（详见右侧「任务面板」）';
    if (sc.id === 'STATIC_A')
      return '已生成提交清单与时间规划，见「任务面板」。';
    return '已记住你的写作风格偏好，之后默认沿用。';
  }
```

Append to `css/styles.css`:

```css
.card-slot:empty{display:none;}
.pcard{margin-top:12px;border:1px solid var(--brand);background:linear-gradient(180deg,#f7faff,#fff);border-radius:var(--radius);padding:14px;animation:cardin .35s ease;}
@keyframes cardin{from{opacity:0;transform:translateY(8px);}to{opacity:1;transform:none;}}
.pcard-head{display:flex;justify-content:space-between;align-items:center;font-weight:700;gap:8px;}
.why-btn{border:none;background:var(--brand-soft);color:var(--brand);border-radius:99px;padding:4px 10px;font-size:12px;cursor:pointer;}
.pcard-body{font-size:14px;line-height:1.6;margin:10px 0;}
.pcard-why{background:#fbfcfe;border:1px dashed var(--line);border-radius:10px;padding:10px 12px;margin-bottom:10px;font-size:13px;}
.pcard-why ul{margin:0 0 8px;padding-left:18px;} .pcard-why li{margin:3px 0;}
.pcard-actions{display:flex;gap:8px;flex-wrap:wrap;}
.flash{animation:flash 1s ease;} @keyframes flash{0%{background:#fffbe6;}100%{background:transparent;}}
```

- [ ] **Step 2: 实现反馈微动效占位（flashAcceptance）**

在 `js/ui.js` 的 `onCardAction` 之后插入（真正动效在 Task 10 完善任务面板后生效，这里先保证函数存在）:

```js
  function flashAcceptance() {
    var p = $('#perception');
    if (p) { p.classList.add('flash'); setTimeout(function(){ p.classList.remove('flash'); }, 1000); }
  }
```

- [ ] **Step 3: 浏览器手动验证**

双击 `index.html`，连点「▶ 下一步」到第 3 回合。
Expected: 聊天区下方滑出蓝色主动卡片，标题“我发现你可能正在准备一个复杂任务”。点「💡 为什么提示我」展开 4 条信号解释。点「稍后提醒」卡片消失且右侧打扰成本上升、tier 回落（演示“刚拒绝就压制”）。重置后点到第 3 回合再点「生成方案框架」，卡片消失、元宝追加“方案框架”回复、感知面板闪一下黄光。

- [ ] **Step 4: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: proactive card with why-explainer and accept/dismiss/mute actions"
```

---

## Task 9: 任务面板与反馈微动效（ui）

**Files:**
- Modify: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `currentScenario().tasks`, `state.accepted`。
- Produces: 就地替换 `renderTasks()`；接受卡片后任务面板逐条填入并高亮（反馈学习的可见微动效）。

- [ ] **Step 1: 实现 renderTasks**

在 `js/ui.js` 中，将占位 `function renderTasks() {}` 整体替换:

```js
  function renderTasks() {
    var ul = $('#task-list'); if (!ul) return;
    var sc = currentScenario();
    // 未接受前：任务面板为空（强调“主动整理”发生在用户接受之后）
    if (!state.accepted) { ul.innerHTML = '<li class="muted">（接受主动建议后，元宝会在此自动整理任务）</li>'; return; }
    var statusZh = { done:'已完成', doing:'进行中', todo:'未开始' };
    ul.innerHTML = sc.tasks.map(function (t, i) {
      return '<li class="task task-' + t.status + '" style="animation-delay:' + (i * 90) + 'ms">'
           + '<span class="dot"></span><span class="task-label">' + t.label + '</span>'
           + '<span class="task-status">' + statusZh[t.status] + '</span></li>';
    }).join('');
  }
```

Append to `css/styles.css`:

```css
.task-list{list-style:none;padding:0;margin:8px 0;display:flex;flex-direction:column;gap:8px;}
.task{display:flex;align-items:center;gap:10px;padding:10px 12px;border:1px solid var(--line);border-radius:10px;animation:taskin .4s ease both;}
@keyframes taskin{from{opacity:0;transform:translateX(-8px);}to{opacity:1;transform:none;}}
.task .dot{width:8px;height:8px;border-radius:99px;background:var(--muted);}
.task-done .dot{background:var(--ok);} .task-doing .dot{background:var(--warn);}
.task-label{flex:1;font-size:14px;} .task-status{font-size:12px;color:var(--muted);}
.task-done .task-label{color:var(--muted);text-decoration:line-through;}
```

- [ ] **Step 2: 浏览器手动验证**

双击 `index.html`。先切到「任务面板」标签——应显示空态提示。回到「对话」，推进到第 3 回合并点「生成方案框架」，再切到「任务面板」。
Expected: 4 条任务自上而下逐条滑入（错峰动画），状态点分别为绿(已完成)/橙(进行中)/灰(未开始)。切到 STATIC_A 场景重复，任务变为该场景的 4 条。

- [ ] **Step 3: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: task panel auto-populates on acceptance with staggered animation"
```

---

## Task 10: 主动服务设置页（ui）

**Files:**
- Modify: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `state.proactivityLevel`。
- Produces: 新增 `renderSettings()`（在 bootstrap 调用）；设置项：主动程度、上下文授权三级、触达方式、敏感内容默认不分析、一键清除记忆。改“主动程度”与感知面板滑块双向同步。

- [ ] **Step 1: 实现 renderSettings**

在 `js/ui.js` 中 `renderTasks` 之后插入:

```js
  function renderSettings() {
    var box = $('#settings-body'); if (!box) return;
    box.innerHTML =
      '<div class="set-group"><div class="set-title">主动程度</div>'
      + '<div class="seg" id="seg-level">'
      + ['off','low','standard','active'].map(function (l){
          return '<button data-level="' + l + '"' + (l===state.proactivityLevel?' class="on"':'') + '>' + levelZh(l) + '</button>';
        }).join('') + '</div>'
      + '<div class="muted set-hint">默认仅在对话内出现，不发系统通知。</div></div>'

      + '<div class="set-group"><div class="set-title">允许使用的上下文（授权分级）</div>'
      + '<label class="chk"><input type="checkbox" checked disabled> 当前会话（基础）</label>'
      + '<label class="chk"><input type="checkbox" checked> 历史对话与偏好</label>'
      + '<label class="chk"><input type="checkbox"> 上传文件 / 日程</label></div>'

      + '<div class="set-group"><div class="set-title">触达方式</div>'
      + '<label class="chk"><input type="checkbox" checked> 对话内卡片</label>'
      + '<label class="chk"><input type="checkbox"> 侧边栏</label>'
      + '<label class="chk"><input type="checkbox"> 系统通知（默认关闭）</label></div>'

      + '<div class="set-group"><div class="set-title">敏感内容</div>'
      + '<label class="chk"><input type="checkbox" checked disabled> 医疗 / 财务 / 情感 / 隐私文件默认不主动分析</label></div>'

      + '<div class="set-group"><button class="ghost" id="btn-clear-mem">一键清除记忆</button></div>';

    Array.prototype.forEach.call(box.querySelectorAll('#seg-level button'), function (b) {
      b.addEventListener('click', function () {
        state.proactivityLevel = b.dataset.level;
        renderSettings(); refreshPerception(); maybeFireCard();
      });
    });
    var clr = $('#btn-clear-mem');
    if (clr) clr.addEventListener('click', function () {
      state.accepted = false; cardMuted = false; resetApp();
      alert('已清除本次演示的记忆与主动状态。');
    });
  }
```

在 `js/ui.js` 的 `DOMContentLoaded` 回调中，把 `renderTasks();` 那一行改为:

```js
    bindUI(); renderChat(); refreshPerception(); renderTasks(); renderSettings();
```

Append to `css/styles.css`:

```css
.set-group{margin-bottom:18px;} .set-title{font-weight:600;margin-bottom:8px;}
.set-hint{font-size:12px;margin-top:6px;}
.seg{display:inline-flex;border:1px solid var(--line);border-radius:10px;overflow:hidden;}
.seg button{border:none;background:#fff;padding:8px 16px;cursor:pointer;border-right:1px solid var(--line);}
.seg button:last-child{border-right:none;} .seg button.on{background:var(--brand);color:#fff;}
.chk{display:block;margin:6px 0;font-size:14px;}
```

- [ ] **Step 2: 浏览器手动验证**

双击 `index.html`，切到「主动服务设置」。
Expected: 看到分段“主动程度”（当前“标准”高亮）、三级上下文授权、触达方式、敏感内容说明、清除记忆按钮。点“积极”后切到对话推进，更易触发卡片；右侧感知面板滑块也应反映为“积极”。点“一键清除记忆”后整个 Demo 重置。

- [ ] **Step 3: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: settings view with proactivity level, graduated permissions, privacy controls"
```

---

## Task 11: 时间快进与紧急提醒彩蛋（ui）

**Files:**
- Modify: `js/ui.js`
- Modify: `css/styles.css`

**Interfaces:**
- Consumes: `currentScenario`, `state`。
- Produces: `fastForward()` 绑定到 `#btn-fast`；演示跨会话主动——快进后追加“紧急”卡片 + 彩蛋文案。

- [ ] **Step 1: 实现 fastForward**

在 `js/ui.js` 中 `renderSettings` 之后插入，并在 `bindUI()` 里追加绑定:

```js
  function fastForward() {
    var sc = currentScenario();
    if (sc.id !== 'MAIN') { alert('时间快进用于演示主线场景的跨会话主动，请切到「主线」。'); return; }
    if (!state.accepted) { alert('请先在第 3 回合接受“生成方案框架”，再演示数日后的主动回访。'); return; }
    pushMessage('ai', '⏰（3 天后）我回来找你了——距提交截止还有 17 天，但你的「Demo 主流程」还没动工。要现在补齐吗？');
    renderChat();
    var slot = $('#card-slot');
    slot.innerHTML =
      '<div class="pcard urgent">'
      + '<div class="pcard-head">⏰ 截止临近提醒'
      + '<button class="why-btn" id="ff-why">💡 为什么现在提示</button></div>'
      + '<div class="pcard-body">距 2026-07-19 截止还有 17 天，关键任务「制作 Demo 主流程」仍为未开始。建议本周内补齐。</div>'
      + '<div class="pcard-why" id="ff-whybox" hidden><ul>'
      + '<li>高时效性：临近已知截止节点</li><li>关键路径任务停滞（未开始）</li>'
      + '<li>这类提醒只在强时效风险时出现，平时沉默</li></ul></div>'
      + '<div class="pcard-actions"><button class="primary" id="ff-ok">现在补齐</button>'
      + '<button class="ghost" id="ff-snooze">明天提醒</button></div>'
      + '<div class="easter">' + sc.easterEgg + '</div></div>';
    $('#ff-why').addEventListener('click', function(){ var w=$('#ff-whybox'); w.hidden=!w.hidden; });
    $('#ff-ok').addEventListener('click', function(){
      pushMessage('ai','好，我们从「Demo 主流程」开始。已把它标为进行中。');
      currentScenario().tasks[2].status = 'doing'; renderChat(); renderTasks(); clearCard();
    });
    $('#ff-snooze').addEventListener('click', clearCard);
  }
```

在 `bindUI()` 函数体内追加一行:

```js
    $('#btn-fast').addEventListener('click', fastForward);
```

Append to `css/styles.css`:

```css
.pcard.urgent{border-color:var(--urgent);background:linear-gradient(180deg,#fff6f7,#fff);}
.easter{margin-top:12px;padding-top:10px;border-top:1px dashed var(--line);font-size:13px;color:var(--brand);font-style:italic;}
```

- [ ] **Step 2: 浏览器手动验证**

双击 `index.html`，主线推进到第 3 回合 → 点「生成方案框架」 → 点「⏩ 时间快进」。
Expected: 元宝追加“3 天后回访”消息；出现红色“截止临近提醒”卡片，可展开“为什么现在提示”；底部以斜体蓝字显示彩蛋句。点「现在补齐」后任务面板“制作 Demo 主流程”变为进行中（橙点）。未接受就快进会弹提示要求先接受。

- [ ] **Step 3: 提交**

```bash
git add js/ui.js css/styles.css
git commit -m "feat: time fast-forward, urgent deadline reminder and self-referential easter egg"
```

---

## Task 12: 收尾——README、演示脚本与端到端走查

**Files:**
- Modify: `README.md`
- Modify: `js/ui.js`（仅必要的 bug 修复，无新功能）

**Interfaces:**
- Consumes: 全部既有功能。
- Produces: 可交付的 README + 一次完整手动走查记录。

- [ ] **Step 1: 写 README**

Overwrite `README.md`:

```markdown
# WUCO Proactive Agent

> 从被动问答到主动任务推进的 C 端 Agent 服务中枢 —— 腾讯黑客松「Agent 主动式服务」参赛 Demo。

**核心主张：竞品都是定时批处理简报（ChatGPT Pulse / Anthropic Orbit）或指令驱动 agent（Gemini），WUCO 做的是「会话内实时意图感知触达」。**

## 运行

无需安装、无需后端：**双击 `index.html`** 即可在浏览器打开。

## 演示脚本（≤4 分钟）

1. 「▶ 下一步」推进对话：问句从“是什么”（认知）→“怎么搭框架”（方法）→“截止时间/怎么推进”（行动）。
2. 看右侧**感知面板**：意图阶段爬升、检测到截止、触发分实时升高。
3. 第 3 回合触发分越过阈值，弹出**主动卡片**；点「💡 为什么提示我」展示判断依据。
4. 点「生成方案框架」：元宝整理结构、**任务面板**自动填充（反馈微动效）。
5. 点「⏩ 时间快进」：演示**跨会话主动**——数日后带着“截止临近+关键任务停滞”回访；结尾彩蛋点题。
6. 拖动感知面板「主动程度」或在「设置」里切「关闭/积极」，现场展示**用户控权**与触达即时变化。

## 设计要点

- **触发评分模型**（源自 Horvitz CHI'99 期望效用）：`score = 需求×时效×置信×接受 ×100 − 打扰成本×40 − 0.5×自发切换惩罚×100`。
- **意图爬升轨迹**触发，明确**排除 idle/定时触发**（CHI'25 证其不可靠）。
- **可解释**：每张卡片可展开“为什么”。**隐私分级 + 默认低打扰 + 敏感场景不主动**。

## 测试

纯逻辑层（`js/engine.js`、`js/scenarios.js`）零依赖单测：

\`\`\`
node tests/engine.test.js
node tests/scenarios.test.js
\`\`\`

## 结构

- `index.html` / `css/styles.css` —— 界面
- `js/engine.js` —— 意图分类 / 信号提取 / 触发评分（纯逻辑，可单测）
- `js/scenarios.js` —— 场景脚本与状态机
- `js/ui.js` —— 渲染与交互
- `docs/superpowers/specs/` —— 设计文档　`docs/superpowers/plans/` —— 实现计划
```

- [ ] **Step 2: 跑全部单测确认绿**

Run: `node tests/engine.test.js && node tests/scenarios.test.js`
Expected: 两个文件分别打印 `ALL PASS (...)`，退出码 0。

- [ ] **Step 3: 端到端手动走查（按 README 演示脚本逐条过）**

双击 `index.html`，完整走一遍演示脚本 1→6，并额外验证：
- 切换三个场景下拉，各自卡片/任务正确；
- 「主动程度=关闭」时全程无卡片；
- 「不再提示此类」后该场景不再弹卡片，「一键清除记忆」可恢复；
- 控制台全程无报错。
Expected: 全部符合。若有异常，在本任务内就地修复 `js/ui.js` 后重走查。

- [ ] **Step 4: 提交**

```bash
git add README.md js/ui.js
git commit -m "docs: add README with demo script; final end-to-end polish"
```

---

## Self-Review

**Spec coverage：**
- §3 五层机制 → 感知(Task7)/预判(Task2-3)/触达判断(Task4)/触达(Task8)/反馈(Task9 微动效) ✓
- §4 触发评分模型（含 λ 自发切换惩罚、Horvitz 注释、排除 idle）→ Task4 ✓
- §5 场景：主线 + 2 静态 + 彩蛋 → Task5/8/11 ✓
- §6 双击运行 / 时间快进 → Global Constraints / Task11 ✓
- §7 四界面 + 可交互感知面板 → 对话(Task6)/任务(Task9)/设置(Task10)/感知(Task7) ✓
- §9 隐私：默认低打扰 / 授权分级 / 可解释 / 可关闭撤回 / 敏感不主动 → Task8/10 ✓

**Placeholder scan：** 无 TBD/TODO；UI 占位函数在 Task6 明确声明、并在 Task7/8/9 就地替换为完整实现，非空泛占位。

**Type consistency：** `analyze/deriveFactors/decideTier/computeTriggerScore` 签名跨 Task4→Task5/7/8 一致；`state` 字段（`history/accepted/proactivityLevel/recentlyRejected`）全程同名；`currentScenario().card.{title,body,why,actions}` 与 Task5 数据结构一致；`levelZh/numToLevel/levelToNum` 在 Task7 定义、Task8/10 复用。

> 已知调参点：Task4 Step4 与 Task5 Step4 注明——若阈值/分值致 tier 不符预期（首回合非 silent 或末回合非 card），按注释微调 `THRESHOLDS.standard` / `STAGE_SCORE` 后重跑测试，这是预期的数值校准而非缺陷。
