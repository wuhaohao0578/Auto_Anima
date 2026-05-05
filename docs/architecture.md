# Architecture & Audit Context

> **目的**: 这份文档是给"接手这个项目的下一个 Claude / 人 / 工具"看的完整上下文。
> CLAUDE.md 是日常协作指令,这份是**架构与设计决策的真相源**。
>
> **配合阅读**: 先读 `CLAUDE.md`(项目指令、协作规则),再读这份(架构、决策、待审清单)。

---

## 0. 项目一句话

**PbD UI Animation Tool** —— 用 LLM + 直接操作的混合范式,把"动画意图"翻译成"生产级前端动画代码"。

- **输入**: 自然语言(v0)→ 视频/关键帧(v3+)
- **核心**: 动画 IR (JSON Schema 描述)
- **输出**: Framer Motion(v0)→ GSAP / CSS(v2)→ Lottie(v3)
- **交互**: 自然语言粗调 + 点元素属性面板精调(v1+)

---

## 1. 系统架构(4 层)

```text
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 1: USER DEMONSTRATION                                         │
│   ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐       │
│   │ Natural Lang│    │ Click + Drag │    │ Keyframe / Video │       │
│   │  ("bounce   │    │  on element  │    │  (v3+, 暂不做)   │       │
│   │   in")      │    │ (v1 Inspector│    │                  │       │
│   └──────┬──────┘    └──────┬───────┘    └────────┬─────────┘       │
└──────────┼──────────────────┼──────────────────────┼────────────────┘
           ▼                  ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 2: INTENT EXTRACTION (LLM-driven)                             │
│      Claude API + structured prompts + JSON-Schema constraints       │
│      Output: candidate intents (Top-K), each scored                  │
│                                                                      │
│      🎯 Contribution ①: prompt design + schema-conformance evaluation│
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 3: ⭐ ANIMATION IR (paper centerpiece) ⭐                    │
│                                                                      │
│   {                                                                  │
│     target:    { selector, role },     ← 谁在动                      │
│     trigger:   { type, params },       ← 什么时候触发                │
│     transform: [ keyframes ],          ← 怎么动                      │
│     timing:    { duration, easing },   ← 节奏                        │
│     stagger:   { children, delay },    ← 群组节奏                    │
│     source:    { type, raw }           ← 元数据(为 v3 视频输入预留)│
│   }                                                                  │
│                                                                      │
│   🎯 Contribution ②: 形式化 IR + 完备性论证 + round-trip 测试        │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 4: MULTI-TARGET COMPILER                                      │
│  ┌────────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐      │
│  │ Framer Mot.│  │   GSAP   │  │ CSS @kfs  │  │ Lottie JSON  │      │
│  │  (v0)      │  │  (v2)    │  │  (v2)     │  │  (v3)        │      │
│  └─────┬──────┘  └────┬─────┘  └─────┬─────┘  └──────┬───────┘      │
│                                                                      │
│  🎯 Contribution ③: 一份 IR → 多套生产代码,fidelity 量化对比        │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 5: PbD FEEDBACK LOOP (v1+)                                    │
│                                                                      │
│   渲染结果 ─► 用户点元素 ─► Inspector 面板调参 ─► AST 改写源码      │
│                  ▲                                       │            │
│                  └───────────── HMR 热更新 ◄─────────────┘            │
│                                                                      │
│   核心机制(借鉴 open-slide,详见 §6):                              │
│   • Vite 插件注入 data-anim-loc="line:col"                          │
│   • Babel parser/types 做源码外科手术                                │
│   • 编辑期 CSS 冻结动画                                              │
│                                                                      │
│   🎯 Contribution ④: NL 粗调 + 直接操作精调的混合 PbD 反馈环        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 已锁死的设计决策(不要重新讨论)

复述自 CLAUDE.md,补充背景:

1. **IR 用 JSON Schema,不自定义 DSL** —— LLM 输出 JSON 最稳定,审稿人也不用学新语法
2. **v0 不做账号 / 云端** —— 本地跑通先,缩小 surface area
3. **输出格式优先级**: Framer Motion > CSS > GSAP > Lottie —— 先证明 web/React 闭环
4. **不做视频理解直到 v0 自然语言路径跑通** —— 避免一开始啃硬骨头
5. **JSX 是输出宿主,不是中心表征** —— IR 才是 single source of truth
6. **复用 open-slide 的 Vite 插件机制**(MIT 协议,允许)—— 不重复造轮子,论文里 cite 即可

---

## 3. 待决定的设计决策(等你做)

### ADR-001: IR 是不是 single source of truth?
- **选项 A**: IR 是真理源,源码是它的投影 → 干净,但每次改都要重编译
- **选项 B**: 源码是真理源,IR 是它的解析视图(open-slide 模式)→ 快,但 IR 会过期
- **选项 C**: 双向同步 → 需要冲突解决协议
- **倾向**: A
- **影响**: 整个 Inspector 编辑流的实现路径

### ADR-002: 项目要不要叫 PbD?
- **选项 A**: 保留 PbD 标签 → 但 v0 自然语言不算真 demonstration,审稿人会挑(Myers 派术语严谨)
- **选项 B**: 改名 "hybrid LLM-assisted animation authoring" → 安全,但失去 PbD 学术血缘
- **倾向**: 待定 —— 取决于 v2 能否补上"用户演示轨迹 → 系统泛化"的真 PbD 通道
- **影响**: paper 的 framing、related work 的写法

### ADR-003: Preview Runtime 在哪?
- **选项 A**: 同一个 React app → 简单,但 HMR 重载会打断动画
- **选项 B**: iframe 沙箱,主 app 派发 IR,iframe 渲染 → 隔离干净
- **选项 C**: 独立 canvas → 失去真实 DOM 上下文
- **倾向**: B
- **影响**: 工具整体形态

### ADR-004: 持久化在哪?
- **选项 A**: 直接写进用户 JSX → git diff 爆炸
- **选项 B**: sidecar `.anim.json`(每个组件一个)→ 可独立 diff/分享
- **选项 C**: 集中 manifest → 复杂
- **倾向**: B

### ADR-005: 动画指令编辑后,如何处理已有的 motion props?
即用户的 `<div>` 要被升级成 `<motion.div>`,且 `transition={CONFIG}` 是变量引用时怎么办?
- 必须列一份 **AST 改写 case 矩阵**,明确哪些场景支持
- 详见 §7 审计 #8

---

## 4. 审计发现(2026-05-05,14 项,按严重度)

### 🔴 严重(v0 启动前必须解决)

#### #1. IR 是空头支票
- 现状: 5 字段(target/trigger/transform/timing/stagger),每个语义都没定义
- 需要: 写 `docs/ir-spec.md v0.1`,带 5 个具体例子的 JSON
  - fade-in
  - slide-from-left
  - stagger list reveal
  - scroll-trigger fade
  - hover sequence (icon scale + label slide)
- 这是 paper 中心,**不能含糊**

#### #2. 多目标编译有不可调和的抽象级差
| Target | 模型 | 适合表达 |
|---|---|---|
| Framer Motion | 组件状态机 | UI 状态切换 |
| GSAP | 命令式时间线 | 复杂时序 |
| CSS @keyframes | 静态关键帧 | 简单循环 |
| Lottie | 矢量图层关键帧 | 装饰动画 |
- IR 必须选**并集 + 有损编译警告**(不能选交集,论文太弱)
- 需要: `docs/compile-fidelity.md` 列每个 target 哪些 IR 特性有损

#### #3. 三处状态同步死结
动画同时存在于: IR JSON / 源码里的 motion props / 运行时 DOM
- 必须先解 ADR-001,定 single source of truth
- 否则 v1 会卡一个月

#### #4. PbD 的 "D" 还没定义
- 自然语言不是 demonstration,是 specification
- 真 PbD 需要"用户给一个例子,系统泛化",目前没有这个
- 必须先解 ADR-002,选 framing
- 不解决 → 投 paper 时被审稿人点名

### 🟡 架构性风险(v1 之前解决)

#### #5. 没有 Preview Runtime 设计 → ADR-003

#### #6. LLM 调用的成本/延迟无规划
- 每次"试一下" = 一次 API 调用,Sonnet 4.6 约 2-5 秒
- PbD 要求快速迭代,这不够快
- 需要: 缓存 / 流式输出 / 本地规则 fallback
- v0 不做也行,但 v1 用户研究时会拉低 SUS 分

#### #7. 持久化层缺失 → ADR-004

#### #8. AST 改写的边界条件
- 用户写 `<div>` 要升级 `<motion.div>` (改 import + 改 tag)
- 已经是 `<motion.div>` 但没 `transition`,要插入新属性
- `transition={CONFIG}` 是变量引用怎么办?
- 需要: `docs/ast-rewrite-cases.md` 列改写矩阵

### 🟢 战略性洞(v2 之前考虑)

#### #9. "为什么不是 Cursor for animations"——最大替代品
2026 年 Cursor / Claude Code / v0 / Bolt 都能"描述 → 动画代码"。差异化:
- 它们是**单次生成**,你是**迭代精修**(PbD 闭环)
- 它们输出**自由代码**,你输出**结构化 IR + 多目标**
- 它们没有**直接操作 + NL 混合**的交互模型

这一段 framing 现在就要想好。

#### #10. 评估方法学未设计
- Task: 实现一个产品页 hero section 进入动画(需要更具体)
- Baseline: "纯 Cursor + Framer Motion 文档" vs "你的工具"
- Metrics: 完成时间 / 最终代码 LOC / SUS / 半结构化访谈
- 现在就要想,因为它会反过来约束 v0-v1 必须支持哪些 feature

#### #11. v3 视频输入会让 v2 的 IR 不够用
- IR 至少要预留 `source: { type, raw }` 元数据 placeholder
- 即使 v0 只填 `'nl'`,也要留空间

### ⚪ 小问题

- #12. CLAUDE.md 写 "Phase: Week 1" 但实际还没启动 → 日期/进度要真实化
- #13. 依赖列表没写 `@babel/parser` / `@babel/types` → v1 必备
- #14. 没提 `react-error-boundary` / telemetry → LLM 生成的代码必有错误,要包边界 + 记录失败 prompt

---

## 5. 论文策略

### 投稿目标(主线只盯三个)

| 优先级 | 会议 | 类型 | 截稿 | 你能投到的最早时间 |
|---|---|---|---|---|
| ⭐⭐⭐ | **CHI LBW** | 4 页 + 海报 | 每年 1 月 | v0-v1 之间(月 2-3) |
| ⭐⭐⭐ | **UIST Demo** | 2 页 + 现场 demo | 每年 6 月 | v1 完成 |
| ⭐⭐⭐ | **IUI 全文** | 12 页 | 每年 1 月 | v2 完成(月 6) |

备选:CHI EA / UIST 正会 / DIS / TOCHI

### 4 个 contribution 怎么挂 paper

| 贡献 | 在哪个 section | 怎么 evaluate |
|---|---|---|
| ① **LLM intent extraction with schema constraints** | System §3 | JSON 合规率、Top-K 命中率、消融 |
| ② **Animation IR (formalization)** | System §4 + Appendix | 表达完备性 + round-trip 保真度 |
| ③ **Multi-target compiler** | System §5 | 跨目标渲染 pixel diff、文件体积、性能 |
| ④ **Hybrid PbD feedback loop** | Interaction §6 | 用户研究、SUS、半结构化访谈 |

任意三个 + 用户研究 = IUI 全文。前两个 + demo 视频 = CHI LBW。

### 路线图与投稿窗口

```text
v0  Month 1   ─ NL → LLM → IR → Framer Motion → 渲染          (论文动作:无)
v1  Month 2-3 ─ + Inspector + Top-K + AST 改写                 (投 CHI LBW / UIST Demo)
v2  Month 4-6 ─ + IR 形式化 + GSAP/CSS + 用户研究 n=8-12       (投 IUI 全文 ⭐)
v3  Month 7-12─ + 视频输入 + Lottie + 大型对照实验 n=20+       (投 CHI / UIST 正会)
```

---

## 6. 与 Prior Work 的清晰 Delta

| 工作 | 它做了什么 | 你的 delta |
|---|---|---|
| **DynaVis** (CHI 2024 Best) | 点 + LLM 改图表 | 你做 UI 动画,不是数据可视化;输出多目标可执行代码 |
| **LogoMotion** (CHI 2025) | LLM 给 logo 生成动画 | 你做交互组件(可被点击调参),有 PbD 迭代环 |
| **Draco** (CHI 2014, Kazi) | 关键帧 + 约束求解 | 你用 LLM 替代约束求解,可处理 NL 意图;有多目标编译 |
| **Mixed-Initiative Animation** (UIST 2018) | 系统建议 + 用户挑选 | 你的 LLM Top-K + Inspector 是新一代实现;落到生产代码 |
| **open-slide** (engineering, 2024) | 点元素改样式 → AST | 你做动画语义,不是静态样式;有 IR 中间层 + LLM 意图提取 |

**关键警告**: LogoMotion 是 CHI 2025,极近,必须明确 delta;DynaVis 是 Best Paper,审稿人会拿来对比。

---

## 7. open-slide 借鉴(infrastructure,不是 contribution)

### 三个必读文件(MIT 协议,可 fork)

- `packages/core/src/app/components/inspector/inspector-panel.tsx` (929 行)
  完整 shadcn 属性面板:Slider / Select / ToggleGroup / 调色盘
- `packages/core/src/app/lib/inspector/fiber.ts` (74 行)
  DOM → 源码行列号双保险:Vite 注入 attr + React Fiber 兜底
- `packages/core/src/vite/loc-tags-plugin.ts`
  编译期注入 `data-slide-loc="line:col"`

### 关键依赖映射

```
open-slide 用的         →  你也用的
@babel/parser+types     →  AST 改写引擎
@vitejs/plugin-react    →  自动加 _debugSource fiber 元数据
radix-ui + shadcn       →  面板 UI
EDITING_FREEZE_CSS      →  编辑期冻结动画(你更需要)
```

### Vite 插件的真实血缘(论文要写清)

```
React 官方 @babel/plugin-transform-react-jsx-source (2015-)
    ├─► click-to-component (2022)
    ├─► react-dev-inspector (12k★)
    ├─► locator.js (商业)
    └─► open-slide loc-tags-plugin (2024)  ← 我们 fork 这个
```

**这一块归 §3 System Implementation,引用即可,不 claim**。

---

## 8. 闭源工具对接策略(v3+)

不直接写二进制文件,三种姿势:

| 工具 | 姿势 | 例子 |
|---|---|---|
| **Lottie** | 直接生成 JSON | IR → Lottie JSON,任意 Lottie 播放器读取 |
| **Rive** | 生成 SDK 调用代码 | 输出 React + `useRive()`,引用 .riv 文件 |
| **After Effects** | 生成 ExtendScript 脚本 | 输出 .jsx 脚本,AE 自己执行后建合成 |

**Lottie 是开放标准**(Airbnb,JSON spec),不算闭源,优先做。

论文叙事:
> *"By specifying animations in a substrate-independent IR, our system supports compilation to multiple target formats with heterogeneous integration strategies: (1) direct code generation for React/CSS/GSAP, (2) declarative format generation for Lottie's open JSON spec, and (3) scripting interface generation for closed tools like After Effects via ExtendScript."*

---

## 9. 技术栈(对应 CLAUDE.md,补充版本)

```
框架:        React 18 + TypeScript + Vite                 (与 open-slide 一致)
动画库:      Framer Motion (主) + GSAP (v2 加入)
LLM:         @anthropic-ai/sdk                           (Claude Sonnet 4.6 默认)
样式:        Tailwind v4 (后期) / 内联 style (早期)
包管理:      pnpm
代码格式:    Biome
AST(v1+):   @babel/parser ^7.29 + @babel/types ^7.29
UI 控件(v1+): radix-ui + shadcn
错误边界:    react-error-boundary
日志:        简单文件日志 + 失败 prompt 收集(v1)
```

---

## 10. 给本地 superpower 审查的提问清单

如果你在本地用 superpower 跑 brainstorm/audit,可以喂这些问题:

### 关于 IR 设计
1. 5 字段 IR(target/trigger/transform/timing/stagger/source)够不够?有没有遗漏的核心动画概念?
2. 怎么表达"基于物理的动画"(spring, gravity)?这个能塞进 timing 还是要新字段?
3. IR 怎么处理"嵌套触发"(父元素 hover → 子元素动画 stagger)?
4. round-trip 测试(代码 → IR → 代码 是否等价)的具体方法?

### 关于 LLM Pipeline
5. JSON-Schema constrained generation 用 Anthropic SDK 怎么做最稳?(tools? structured output?)
6. Top-K 候选怎么打分?LLM self-rank 还是另起一个 evaluator?
7. 失败重试策略:LLM 输出不合规时,是 repair 还是重新生成?

### 关于交互
8. NL 粗调 + Inspector 精调,两条编辑路径如何同步状态?谁是真理源?
9. 编辑期间是否允许新 NL 输入,还是必须先退出编辑模式?
10. 多个动画在同一元素上叠加时(如 hover + scroll),如何 IR 表达?如何 UI 编辑?

### 关于评估
11. v2 用户研究的 task 设计:开放任务还是封闭任务?对照组用什么?
12. 跨目标 fidelity 怎么量化?像素 diff 不够,还要看时序、缓动?
13. PbD 的"learnability"怎么测?用户多少次尝试能掌握?

### 关于位置
14. 和 LogoMotion(CHI 2025)的 delta 还能更锐利吗?他们是否可能扩展到 UI 动画?
15. 和 Cursor / v0 / Bolt 这种 LLM coding 工具,在动画这个垂直领域我们的不可替代性是什么?

---

## 11. 当前项目状态(2026-05-05)

```
代码:          0 行(只有 CLAUDE.md + 这份 architecture.md)
分支:          claude/create-claude-md-7jJAx
近期 commits:  f1e5ba2 docs: add CLAUDE.md project instructions
              260ce17 docs: add open-slide as reference project in CLAUDE.md
下一步:        ① 跑本地 superpower 审查(用本文档作输入)
              ② 解决 ADR-001 ~ ADR-005
              ③ 写 docs/ir-spec.md v0.1
              ④ 启动 v0 pnpm create vite
```

---

## 12. 给下一个接手者的话

如果你是来继续这个项目的 Claude / 人 / 工具:

1. **先读 CLAUDE.md** 知道用户是谁、怎么协作
2. **再读这份** 知道架构、决策、待审清单
3. **不要重新评估全局**,基于已有决策给下一步
4. **挑战已锁决策时,给具体理由**,别只说"我觉得"
5. **优先解决 §4 里的 🔴 严重问题**,其他是后话
