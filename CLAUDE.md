# CLAUDE.md

这个文件是给 Claude Code 的项目指令。每次 Claude Code 在这个项目里工作时,先读这个文件。

---

## 项目是什么

**PbD UI Animation Tool** —— 一个 Programming by Demonstration 范式的 UI 动画工具。

- **输入**:视频 / 关键帧 / 自然语言演示
- **处理**:LLM 理解动画意图 → 结构化中间表示 (IR) → 编译
- **输出**:生产级前端动画代码 (Framer Motion / GSAP / CSS / Lottie)

学术对接:Brad Myers, Sumit Gulwani, Lydia Chilton, Rubaiat Habib Kazi, Maneesh Agrawala 这条 PbD 研究脉络。
关键参考论文:LogoMotion (CHI 2025), DynaVis (CHI 2024 Best Paper), Draco, Motion Amplifiers, Mixed-Initiative Animation, MotionCanvas。

---

## 我是谁

- 名字:Haohao,深圳,30+
- 背景:视觉传达自考在读 (深大),强系统设计直觉 + 审美 + 产品经验
- **编程零基础但开始学**——这一点很重要,影响你和我交流的方式
- 英语中等
- 战略目标:作品集 → 海外硕士 (ITP / RCA / Parsons / KMD / 冲 Media Lab / CMU HCII) → 创业或就业

---

## 你怎么和我协作

### 默认行为
- **直接执行,不要先问"你确定吗"**。我让你建项目就建,让你改代码就改。
- **写代码时优先跑通,再优化**。能 work 的烂代码 > 完美的不能跑的代码。
- **报错先自己尝试解决一轮**,解决不了再问我。
- **每次改完跑一下** (`pnpm dev` 或对应命令),确认没坏。

### 解释代码时
- 我零基础,**默认我什么都不懂**。但不要废话铺垫,直接讲核心。
- 涉及新概念时,用一句话定义 + 一个例子,不要展开成教程。
- 我问"X 是什么"时,先一句话答,再给细节。

### 不要做的事
- 不要劝我"先学基础再做项目"——我就是边做边学。
- 不要每次回复都铺垫"这是个不错的想法"。直接干。
- 不要拆任务到"原子级",我要的是目标导向的下一步。
- 不要假设我做不到。能不能做到做了才知道。
- 不要写"建议你考虑..."这种话。要么给方案,要么直说为什么不行。

### 要做的事
- 给具体可执行的代码 / 命令。
- 当我描述意图时,先帮我落地成代码,再讨论优化。
- 主动指出我代码里的问题 (但简洁,一次一个,不要列清单恐吓我)。
- 帮我搜索资料、论文、工具时信息密度要高。

---

## 技术栈 (v0)

```
前端框架:    React 18 + TypeScript + Vite
动画库:      Framer Motion (主) + GSAP (后期加入)
LLM:        Claude API (@anthropic-ai/sdk)
样式:        Tailwind CSS (后期) / 内联 style (早期够用)
包管理:      pnpm
代码格式:    Biome (轻量,比 ESLint+Prettier 简单)
```

不要引入这些,除非我明确说要:
- Redux / Zustand 等状态管理库 (useState 够用)
- Next.js (Vite 够了,你不需要 SSR)
- 复杂的测试框架 (早期手测就行)
- Storybook 等组件文档工具

---

## 项目结构 (会随进度演进)

```
pbd-lab/
├── CLAUDE.md           ← 你正在读这个
├── README.md           ← 给外人看的
├── docs/
│   ├── ir-spec.md      ← 动画 IR 的设计文档
│   ├── findings/       ← 每天的发现笔记 (day1-findings.md …)
│   └── papers/         ← 论文阅读笔记
├── src/
│   ├── App.tsx
│   ├── lib/
│   │   ├── llm.ts      ← Claude API 调用封装
│   │   ├── ir.ts       ← IR 类型 + schema
│   │   └── compile.ts  ← IR → Framer Motion props
│   └── components/
└── .env.local          ← API key (永远不 commit)
```

---

## 当前阶段 (实时更新)

**Phase**: Week 1 - 打地基
**目标**: 能跑通 "自然语言 → LLM → Framer Motion JSON → 渲染" 的最小闭环

**进度清单** (你帮我维护这个,完成的打 ✅):
- [ ] Day 1: 最小 demo 跑通
- [ ] Day 2: LLM 返回多个候选方案,可微调参数
- [ ] Day 3: 写 day1-3 findings 笔记
- [ ] Week 2: 复刻 DynaVis 核心机制
- [ ] Week 3: 接入动画属性生成
- [ ] Week 4: IR v0 设计完成

---

## 关键设计决策 (做过的不要回头讨论)

1. **IR 用 JSON Schema 描述,不用自定义 DSL**——LLM 输出 JSON 最稳定
2. **v0 不做账号系统、不做云端存储**——本地跑通先
3. **v0 输出格式优先级**: Framer Motion > CSS > GSAP > Lottie
4. **不做视频理解,直到 v0 跑通自然语言路径**——避免一开始就啃硬骨头

这些决策可以挑战,但挑战时给具体理由,别只说"我觉得"。

---

## 论文 / 参考阅读

读过的 (你可以引用):
- LogoMotion (CHI 2025)
- DynaVis (CHI 2024)

待读 (按优先级):
1. Draco (CHI 2014) - Kazi
2. Motion Amplifiers (CHI 2016)
3. Mixed-Initiative Animation (UIST 2018)
4. Motion Vectorization (SIGGRAPH Asia 2023)
5. MotionCanvas (2025)

---

## 参考项目 (源码值得读)

### open-slide (https://github.com/1weiho/open-slide)
开源幻灯片框架,作者 1weiho。和我们 PbD 工具共享同一套核心架构:
**点击 DOM 元素 → 弹出属性面板 → 改值 → AST 改写源码 → HMR 热更新**

到 v1 做"点元素调动画参数"功能时,把这三个文件当教科书读:
- `packages/core/src/app/components/inspector/inspector-panel.tsx` — 完整 shadcn 属性面板 (滑杆/调色盘/下拉/toggle)
- `packages/core/src/app/lib/inspector/fiber.ts` — DOM 元素 → 源码行列号的双保险机制 (Vite 注入 attr + React Fiber 兜底)
- `packages/core/src/vite/loc-tags-plugin.ts` — Vite 插件,编译期给每个 JSX 加 `data-slide-loc="line:col"`

关键依赖映射 (我们要复用):
- `@babel/parser` + `@babel/types` → AST 改写引擎
- `@vitejs/plugin-react` → 自动加 `_debugSource` fiber 元数据
- `radix-ui` + `shadcn` + `Slider`/`Select`/`ToggleGroup` → 面板 UI
- 编辑期间注入 CSS 冻结动画 (`EDITING_FREEZE_CSS`) ← 我们更需要这个

**注意**:open-slide 是工程项目,**没有发表论文**。这意味着如果我们把同样的交互范式用在"动画"语义上、加上正式的用户研究和 IR 形式化,**有论文空间**。

---

## 工作节奏

- 我每天投入 2-3 小时
- 每周日做一次 weekly review
- 每月一次大盘点

每次会话开始时,**直接基于上下文给下一步**,不要重新介绍 PbD、不要重新评估全局。

---

## 一句话总结

**目标导向、直接执行、零废话。我要的是一个能落地的产品 + 一份能进顶校的作品集,不是一份完美的教程。**
