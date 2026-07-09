# Remotion 技术调研与 Harness Agent 剪映落地方案

> **文档性质：** 技术调研汇报文档，面向剪映团队
> **调研日期：** 2026-07-08
> **核心主题：** Remotion 的使用方法、原理与技术实现，以及如何通过 Harness Agent 工程将其接入剪映 AI 视频编辑程序，实现生成级可落地的视频生产方案

---

## 目录

1. [执行摘要](#section-1)
2. [Remotion 技术原理与实现细节](#section-2)
3. [Remotion 与剪映架构的契合分析](#section-3)
4. [Harness Agent 工程总体架构](#section-4)
5. [Harness 核心模块详解](#section-5)
6. [剪映落地方案与实施计划](#section-6)
7. [成本与商业分析](#section-7)
8. [风险评估与应对策略](#section-8)
9. [附录：参考来源](#section-9)

---

<a id="section-1"></a>

## 一、执行摘要

### 1.1 核心结论

**Remotion 可以作为剪映 AI 视频生成能力的程序化渲染引擎，通过 Harness Agent 工程实现可落地的生成级视频生产。** 具体而言：

- **Remotion 是什么：** 一个用 React 编写视频的开源框架（52K+ GitHub stars），将视频定义为「React 组件逐帧渲染 + FFmpeg 编码 → MP4」，实现视频的程序化、确定性生成。
- **对剪映的价值：** 剪映当前的 AI 文案成片依赖视频生成模型（Seedance 等），单条 5 秒 1080p 视频成本 ¥3.89。Remotion 方案可将信息型/模板化内容的视频生成成本降至接近零（仅 LLM 代码生成成本 ~¥0.05/条），**成本降低 97%+**。
- **适用场景：** 知识科普、数据可视化、品牌营销、字幕动画、模板化内容——这些场景 Remotion 在成本、可控性、精确性上远优于视频生成模型。需要真实世界画面的场景（美食/旅行/人物）仍需视频生成模型。
- **Harness 工程：** Agent 驱动 Remotion 需要 5 层工程化设施（代码生成管线 + 组件库 + 质量保障 + 性能优化 + 错误恢复），共 15 个 MVP 必须模块。
- **许可证：** Remotion 为 source-available 许可证，公司 ≥4 人需 Company License（$25/开发者席位/月，最低 $100/月），Enterprise License $500/月起。需与 Remotion 团队（hi@remotion.dev）确认纯客户端渲染的计费方式。

### 1.2 关键数据

| 维度 | Remotion 方案 | 视频生成模型（Seedance） | 剪映当前方案 |
|---|---|---|---|
| 单条 30 秒视频成本 | **~¥0.05**（仅 LLM） | ~¥23+ | ¥23+ |
| 1000 条/月成本 | **~¥50** | ~¥23,000+ | ~¥23,000+ |
| 画面可控性 | ✅ 帧级精确 | ❌ AI 抽卡 | ❌ AI 抽卡 |
| 生成一致性 | ✅ 100% 确定性 | ❌ 每次不同 | ❌ 每次不同 |
| 真实画面能力 | ❌ 程序化绘制 | ✅ AI 生成 | ✅ AI 生成 |
| 渲染速度 | 服务器GPU单机 10-30 fps / Lambda 并行 | ~41 秒/5 秒视频 | ~41 秒/5 秒视频 |

---

<a id="section-2"></a>

## 二、Remotion 技术原理与实现细节

### 2.1 Remotion 是什么

Remotion 是一个**用 React 编写视频**的框架，由瑞士开发者 Jonny Burger 于 2020 年创建，Remotion AG 公司全职维护。核心思想：**视频的本质是逐帧画面 + 音轨，React 组件可以声明式地描述任意一帧的 UI 状态**——因此用 React 组件定义每一帧的视觉内容，按帧率驱动渲染，再合成为视频文件。

**版本现状（2026 年 7 月）：** Remotion 4.0.x 系列，GitHub 52K+ stars，每周有 PR 合入，迭代活跃。

### 2.2 核心原理链路

Remotion 的本质链路可以概括为：**React 组件描述每一帧 -> 渲染器逐帧捕获画面 -> 编码器合成视频文件**。对剪映团队而言，关键不是代码写法，而是它为什么适合信息型/模板化视频：画面由确定性代码驱动，成本低、可控、可批量复用。

```
React Composition
声明场景、字幕、图表、动画
        |
        v
Frame Driver
useCurrentFrame() 注入当前帧号
        |
        v
Frame Capture
服务端 Chrome 截图 / 客户端 DOM -> Canvas
        |
        v
Video Encoding
FFmpeg / WebCodecs 合成 MP4
        |
        v
可预览视频 + 可回写草稿元数据
```

#### 阶段 1：React 组件声明帧画面

Remotion 将时间维度引入 React：组件不是响应用户点击，而是响应“当前第几帧”。因此动画必须是 `frame` 的纯函数；同一帧号、同一 props 必须输出同一画面。

```tsx
const Scene = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const opacity = interpolate(frame, [0, 15], [0, 1], {
    extrapolateLeft: 'clamp',
    extrapolateRight: 'clamp',
  });
  return <AbsoluteFill style={{ opacity }}>标题 / 图表 / 素材</AbsoluteFill>;
};
```

| 机制 | 作用 | 对 Agent / 剪映的影响 |
|---|---|---|
| `useCurrentFrame()` | 获取当前帧号 | Agent 生成动画时必须用帧驱动，不能用 CSS transition |
| `interpolate()` | 将帧号映射成透明度、位置、缩放等动画值 | 必须 clamp，避免 opacity 负数、scale 异常 |
| `spring()` | 生成弹簧动画曲线 | 适合标题、卡片入场，但复杂参数应由组件库封装 |
| `<Sequence>` | 控制场景在时间线中的起止位置 | 支持把每个镜头封装成独立组件，再由外层编排 |

#### 阶段 2：帧捕获

| 捕获模式 | 工作方式 | 优点 | 限制 | 建议用途 |
|---|---|---|---|---|
| 服务端 `@remotion/renderer` | Headless Chrome 打开 Composition，逐帧截图 | CSS 支持完整，稳定性高 | 有服务器成本，渲染速度约 1-3 fps | MVP 最终导出和兜底渲染 |
| 客户端 `@remotion/web-renderer` | 浏览器内 DOM -> Canvas -> ImageBitmap | 低云成本，符合 Web-first 方向 | alpha，CSS 子集，受设备性能影响 | POC、低清预览、未来客户端导出 |
| Lambda / 分片渲染 | 将帧范围分发到多个 worker 并行渲染 | 长视频/批量渲染速度快 | 成本和 Remotion Automators 计费需确认 | P2 批量生成或长视频导出 |

服务端路径适合先上线，因为可控、稳定、CSS 覆盖完整；客户端路径与剪映 Web-first 方向一致，但必须先验证浏览器兼容性、CSS 子集和许可证计费。

#### 阶段 3：视频编码

| 编码方式 | 技术栈 | 对剪映的意义 | 风险 |
|---|---|---|---|
| 服务端编码 | FFmpeg / H.264 / AAC | 输出兼容性最好，适合最终成片 | Docker、字体、Chrome/FFmpeg 环境要固定 |
| 客户端编码 | WebCodecs + muxer（如 Mediabunny） | 可降低云成本，支持浏览器内导出 | 浏览器兼容性、内存、长视频稳定性 |
| 混合策略 | 客户端低清预览 + 服务端高清导出 | 兼顾体验、成本和稳定性 | 需要统一帧号、色彩和字体渲染结果 |

**工程结论：** 剪映落地不应一开始押注单一渲染模式。MVP 用服务端渲染保证稳定，客户端渲染作为 POC 和低清预览能力逐步验证；所有渲染模式都从同一份 Timeline DSL / VideoSpec 派生，避免业务逻辑分叉。

### 2.3 核心 API 详解

下表列出 Remotion 的核心 API，并标注 LLM 生成代码时的常见错误：

| API | 作用 | LLM 常见错误 | 正确用法 |
|---|---|---|---|
| `useCurrentFrame()` | 获取当前帧号 | 在 `useMemo` 内调用（违反 hooks 规则） | 必须在组件顶层调用 |
| `useVideoConfig()` | 获取 fps/宽高/总帧数 | 硬编码 fps 而非从 config 获取 | 始终用 `const { fps } = useVideoConfig()` |
| `interpolate()` | 帧值映射到动画值 | 忘记 `extrapolate: 'clamp'`，导致 opacity 变负数 | `interpolate(frame, [0, 30], [0, 1], { extrapolateRight: 'clamp' })` |
| `spring()` | 物理弹簧动画 | 用 CSS `transition` 替代（多线程渲染闪烁） | `spring({ frame, fps, config: { damping: 200 } })` |
| `<Sequence>` | 时间线序列 | 忘记设置 `from`，所有场景同时从帧 0 开始 | `<Sequence from={150} durationInFrames={90}>` |
| `<Composition>` | 定义视频合成 | `durationInFrames` 与实际场景总时长不匹配 | 精确计算所有场景帧数之和 |
| `<Img>` / `<Video>` | 媒体组件 | 用原生 `<img>` / `<video>`（渲染时不等待加载） | 必须用 Remotion 媒体组件 |
| `staticFile()` | 静态文件引用 | 用字符串路径（`src="/assets/logo.png"`） | `staticFile('assets/logo.png')` |
| `random()` | 确定性随机 | 用 `Math.random()`（非确定性，每次渲染不同） | `random('seed-1')` |
| `delayRender()` | 等待异步资源 | 忘记调用 `continueRender()`，导致渲染超时 | `const h = delayRender(); fn().then(() => continueRender(h))` |

### 2.4 能力矩阵

#### 能做什么

| 能力 | 说明 | 成熟度 | 关键包 |
|---|---|---|---|
| **文字动画** | 逐字打字机、逐词高亮、kinetic typography、字符级 stagger | 成熟 | `@remotion/google-fonts` |
| **图形与形状** | SVG 路径动画（`evolvePath()`）、形状生成、渐变背景 | 成熟 | `@remotion/paths`, `@remotion/shapes` |
| **转场特效** | fade / slide / wipe / clockWipe / flip | 成熟 | `@remotion/transitions` |
| **数据可视化** | 图表、数字滚动、进度条、地图 | 成熟（需自行组合 d3 等） | - |
| **动态布局** | 响应式布局、stagger 入场、Spring 物理动画 | 成熟 | `spring()` + `interpolate()` |
| **图片展示** | Ken Burns（缩放平移）、轮播、视差 | 成熟 | - |
| **嵌入真实视频** | `<OffthreadVideo>` 逐帧提取（服务端），`<Video>` WebCodecs（浏览器端） | 成熟 | `@remotion/media` |
| **音频** | TTS 配音、BGM 混音、音量曲线控制 | 成熟 | `<Audio>` |
| **3D 渲染** | Three.js 集成 | 可用（复杂场景慢） | `@remotion/three` |
| **Lottie 动画** | 直接渲染 Lottie JSON | 成熟 | `@remotion/lottie` |
| **噪声/粒子** | Perlin 噪声 | 可用 | `@remotion/noise` |
| **光效** | 电影感光晕 | 成熟 | `@remotion/light-leaks` |

#### 不能做什么

| 限制 | 说明 | 对剪映的影响 |
|---|---|---|
| **生成真实视频内容** | 程序化绘制，无法生成真实世界画面 | 美食/旅行/人物场景仍需 Seedance |
| **复杂 3D 场景** | 每帧截图 Three.js canvas，渲染慢 | 不适合做 3D 动画视频 |
| **实时视频处理** | 先渲染再播放，非实时流处理 | 无法做实时滤镜预览、实时美颜 |
| **浏览器端 CSS 全覆盖** | 客户端渲染不支持 `clip-path`、`backdrop-filter` 等 | MVP 组件需限制 CSS 使用范围 |

### 2.5 渲染模式对比

| 维度 | 服务端 (`@remotion/renderer`) | 浏览器端 (`@remotion/web-renderer`) |
|---|---|---|
| 运行环境 | Node.js + Headless Chrome | 用户浏览器 |
| 帧捕获 | Chrome 截图，CSS 支持完整 | DOM -> Canvas，CSS 子集 |
| 编码 | FFmpeg | WebCodecs + muxer |
| 速度 | 1-3 fps（单机）/ 可分片并行 | 0.5-2 fps，取决于设备 |
| 成本 | 有服务器/Lambda 成本 | 云成本低，但占用用户设备 |
| 成熟度 | 生产可用 | alpha，需要 POC 验证 |
| 建议定位 | MVP 导出主路径 | P2 低清预览和客户端导出探索 |

**客户端 CSS 约束可压成三类：**

| 分类 | CSS / 能力 | 处理建议 |
|---|---|---|
| 可安全使用 | `background`、`border`、`opacity`、`transform`、基础阴影 | 可进入 MVP 组件规范 |
| 谨慎使用 | `filter`、`mask-image`、复杂字体效果 | 需要浏览器兼容测试和视觉回归 |
| 禁用或服务端兜底 | `clip-path`、`backdrop-filter`、`mix-blend-mode`、复杂 3D | 不进入客户端 MVP，必要时切服务端渲染 |

### 2.6 对 Harness 设计的启发

Remotion 官方已经提供 AI 生成相关材料，包括 System Prompt、Dynamic Compilation、`@remotion/eslint-plugin`、Prompt to Motion Graphics 模板和 Agent Skills。这些内容不在本章重复展开，后续 Harness 章节只引用其工程价值：

| 官方能力 | 对本方案的启发 | 在本文中的落点 |
|---|---|---|
| System Prompt | 证明 Remotion 代码生成需要明确规则，而不是自由提示词 | 第 4 部分的 Agent 约束设计 |
| Dynamic Compilation | 证明浏览器内 Babel 编译 + API 注入可行，但需要安全边界 | 第 5 部分的路径 2 / 代码沙箱 |
| ESLint 插件 | 将 LLM 常见错误转成自动化静态检查 | 第 4.5 验证管线、第 5.4 质量保障 |
| Prompt to Motion Graphics 模板 | 提供“描述 -> 代码 -> 预览 -> 修复”的参考闭环 | 第 6 部分的剪映落地任务 |
| Agent Skills | 说明规则可模块化注入，但不能替代产品级 Harness | 第 5 部分的组件注册表和 catalog 设计 |

**结论：** Remotion 已经具备 AI 生成代码的基础设施，但剪映级产品不能直接复用“LLM 生成代码 -> 预览”的 demo 模式。需要在其上增加 DSL、组件注册表、剪映草稿协议、许可证控制、素材授权和多层验证，才能成为商业产品可用的生成链路。

---

<a id="section-3"></a>

## 三、Remotion 与剪映架构的契合分析

### 3.1 剪映现有架构回顾

剪映的 AI 文案成片采用**固定流水线编排**：

```
用户输入话题 → 脚本生成（豆包+DeepSeek）→ 分镜生成（LLM 解析）
→ 素材生成（Seedream/Seedance/OmniHuman）→ 剪辑合成（多轨道时间线）
→ 导出（WASM 引擎）
```

**渲染管线：** 客户端 WASM（C++ → Emscripten，SIMD + WebCodecs 加速）+ 云端 AI 生成（火山方舟 + veFuser）

**草稿协议：** 结构化 JSON（`draft_content.json`），包含 tracks（视频/音频/特效/字幕轨）、materials（素材引用）、segments（时间轴排列）

### 3.2 Remotion 在剪映中的定位

Remotion 不是替代剪映的 WASM 编辑引擎，而是作为**程序化视频合成引擎**补充：

```
┌────────────────────────────────────────────────────────────┐
│                        剪映编辑器 UI                         │
│  Agent 对话 / 生成进度 / 代理预览 / 时间线 / 属性面板            │
└─────────────────────────────┬──────────────────────────────┘
                              │ Intent + Project Context
                              ▼
┌────────────────────────────────────────────────────────────┐
│                     Agent Orchestrator                     │
│  意图理解 → 分镜规划 → 素材策略 → 能力路由 → 失败修复             │
└─────────────────────────────┬──────────────────────────────┘
                              │ Structured Output
                              ▼
┌────────────────────────────────────────────────────────────┐
│                       Agent Video 工程蓝图                  │
│ project / assets / scenes / layers / timing / styles       │
│ constraints / render_policy / native_capability            │
└───────────────┬──────────────────────────────┬─────────────┘
                │                              │
                ▼                              ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│  剪映编辑引擎          │  │  Remotion 合成引擎            │
│  (WASM + WebCodecs)  │  │  (React 组件 → 帧 → 视频)      │
│                      │  │                              │
│  职责：               │  │  职责：                       │
│  · 素材裁剪/拼接       │  │  · 程序化画面生成              │
│  · 多轨时间线实时预览   │  │  · 字幕动画/转场/模板          │
│  · 滤镜/调色           │  │  · 数据可视化                 │
│  · 音频混音            │  │  · AI 生成代码的渲染           │
│  · 本地导出            │  │  · 程序化视频合成              │
│                      │  │                              │
│  处理：真实素材        │  │  处理：程序化内容 + 素材混合      │
└──────────┬───────────┘  └──────────┬───────────────────┘
           │                         │
           └────────────┬────────────┘
                        ▼
              ┌─────────────────┐
              │  统一导出管线     │
              │  WebCodecs 编码  │
              │  FFmpeg (服务端) │
              └─────────────────┘
```

**两个引擎共存：** WASM 引擎处理真实素材的实时编辑，Remotion 处理程序化画面生成和模板渲染，通过统一的草稿协议 JSON 对接。

### 3.3 与剪映草稿协议的对接

剪映使用 `draft_content.json` 作为中间格式。Remotion 方案需要**新增 `programmatic` 轨道类型**：

```json
{
  "tracks": [
    {
      "type": "video",
      "segments": [{ "material_id": "mat_001", "start": 0, "duration": 5.0 }]
    },
    {
      "type": "programmatic",
      "segments": [
        {
          "id": "prog_001",
          "start": 5.0,
          "duration": 3.0,
          "component": "DataChart",
          "props": {
            "title": "用户增长",
            "data": [100, 250, 800, 1500, 3200],
            "chartType": "bar"
          }
        },
        {
          "id": "prog_002",
          "start": 8.0,
          "duration": 4.0,
          "component": "AnimatedTitle",
          "props": {
            "text": "关键发现",
            "animation": "typewriter",
            "fontSize": 72
          }
        }
      ]
    },
    {
      "type": "subtitle",
      "segments": [{ "text": "用户增长 320%", "start": 5.0, "duration": 3.0 }]
    }
  ]
}
```

**转换器职责：**
1. `programmatic` 轨道 → Remotion 组件渲染
2. `video` / `audio` 轨道 → `<OffthreadVideo>` / `<Audio>` 组件
3. `subtitle` 轨道 → 字幕动画组件
4. 合并为完整 Remotion Composition

### 3.4 Remotion vs 视频生成模型：场景适配

| 场景 | 推荐方案 | 理由 |
|---|---|---|
| 知识科普 / 数据可视化 | **Remotion** | 精确数据展示，零成本 |
| 品牌营销 / 产品演示 | **Remotion** | 模板化，可控性强 |
| 新闻资讯 / 字幕视频 | **Remotion** | 文字驱动，动画精确 |
| 教程 / 操作演示 | **Remotion + 截图** | 截图 + 程序化标注 |
| 故事 / 叙事短片 | 视频生成模型 | 需要真实感画面 |
| 美食 / 旅行 / 生活 | 视频生成模型 | 需要真实世界画面 |
| 人物口播 / 数字人 | 视频生成模型 + 数字人 | 需要人脸动态 |

**结论：** Remotion 覆盖剪映 AI 文案成片中约 40-60% 的内容场景（信息型/模板化），这些场景的成本可从 ¥3.89/条降到 ~¥0.05/条。需要真实画面的场景仍用 Seedance 等视频生成模型。

---

<a id="section-4"></a>

## 四、Harness Agent 工程总体架构

### 4.1 从第 3 部分到 Harness 架构：为什么还需要一层工程系统

第 3 部分已经说明 Remotion 在剪映中的定位：它不替代剪映 WASM/WebCodecs 编辑引擎，而是补充“程序化片段生成”能力。这个定位解决了“Remotion 放在哪里”的问题，但还没有解决“AI 怎么稳定地产出可编辑草稿”的问题。

如果只是把 Remotion 直接接入剪映，会遇到四个断点：

| 第 3 部分得到的架构判断 | 直接接入会遇到的问题 | 第 4 部分 Harness 要补的能力 |
|---|---|---|
| Remotion 适合信息型/模板化内容 | Agent 可能生成错误组件、错误时间线、错误素材引用 | 用 Timeline DSL、组件注册表、Schema 校验约束输出 |
| Remotion 与剪映双引擎共存 | 两套引擎缺少统一中间层，草稿可编辑性会断掉 | 用 Draft Patch + VideoSpec 把剪映草稿和 Remotion 渲染连接起来 |
| `programmatic` 轨道可以承载程序化片段 | 片段生成失败后无法定位该重跑脚本、素材、音频还是布局 | 用多 Agent 状态机和局部重跑机制拆分责任 |
| 信息型视频成本低、可控性强 | 如果每次错误都进入完整渲染，成本和等待时间仍然不可控 | 用低成本优先的验证管线拦截错误 |
| 5000+ 人商业产品需考虑许可证和替代路线 | 直接绑定 Remotion 会形成采购和技术锁定 | 用可替换渲染层设计保留 HyperFrames / 自研路线 |

因此，Harness 不是“多包一层 Agent 调用”，而是把第 3 部分的双引擎判断产品化：**剪映负责可编辑时间线，Remotion 负责程序化渲染，Harness 负责把 AI 生成结果变成稳定、可验证、可回滚、可替换的工程链路。**

本方案采用 **“强约束中间层 + 多 Agent 分工 + 分层验证 + 可替换渲染引擎”** 的架构，而不是让一个 Agent 直接生成一段 Remotion 代码。

### 4.2 一个贴近产品的端到端例子

假设用户在剪映里输入：

```text
做一个 30 秒 AI 视频剪辑工具发布短片，风格高级，竖屏，用我们的产品截图和 Logo。
```

没有 Harness 的做法通常是：让 LLM 直接生成一段 Remotion 代码或一个 MP4。这条路 demo 很快，但产品化会出问题：字幕不可编辑、Logo 替换困难、素材来源不可追踪、代码错误难定位，Remotion 许可证或客户端渲染受阻时也难迁移。

Harness 的做法是把这个需求拆成可编辑的视频工程：

```
用户 Prompt
  “30 秒 AI 视频剪辑工具发布短片，竖屏，高级感，用产品截图和 Logo”
        |
        v
Director / Script Agent
  0-3s   品牌标题动画
  3-8s   产品截图 + Ken Burns 动效
  8-15s  三个功能点卡片 + 字幕
  15-22s 成本/效率对比图表
  22-30s Logo + CTA 结尾卡
        |
        v
Timeline DSL
  tracks: image / subtitle / audio / programmatic
  materials: logo、产品截图、TTS、BGM
  aiMeta: 生成来源、版本、可编辑参数
        |
        +-----------------------------+
        |                             |
        v                             v
剪映 Draft Patch                 Remotion VideoSpec
图片轨、字幕轨、音频轨             AnimatedTitle / FeatureCards
智能片段参数可编辑                 DataChart / EndCard
        |                             |
        +-------------+---------------+
                      v
              预览、质检、导出
```

用户最终看到的不是一条死 MP4，而是一条可继续编辑的剪映时间线：可以改标题、替换截图、调字幕、换 Logo、重生成图表片段。这个例子也解释了为什么第 4 部分要引入 Timeline DSL、VideoSpec、Draft Patch 和多层验证。

### 4.3 为什么采用 Timeline DSL 作为核心中间层

整个架构的核心不是 Remotion，而是 `Timeline DSL`。Remotion 只是 DSL 的一个渲染目标；剪映草稿、预览、服务端导出、客户端导出都从 DSL 派生。

```
用户意图 / 素材 / 模板
        |
        v
CreativeBrief / Storyboard / AudioAssets
        |
        v
┌──────────────────────────────────────────────┐
│              Timeline DSL                     │
│ tracks / clips / materials / styles / aiMeta  │
└───────────────┬───────────────────┬──────────┘
                |                   |
                v                   v
        剪映 Draft Patch       Remotion VideoSpec
        原生轨道可编辑          程序化片段可渲染
                |                   |
                v                   v
        剪映编辑器时间线       Player / Renderer / WebCodecs
                \                   /
                 \                 /
                  v               v
                  统一预览与导出管线
```

选择 DSL 的原因：

1. **可编辑性优先。** 剪映是编辑器，不是纯生成器。字幕、图片、音频、转场、程序化片段都必须能回到时间线里继续编辑。
2. **降低 Agent 自由度。** Agent 生成 DSL 比生成代码更稳定，字段可以被 Zod/JSON Schema 检查，失败能精确定位。
3. **降低许可证锁定。** 如果 Remotion Enterprise / Automators 条款不适合 5000+ 人商业产品，同一 DSL 可以映射到 HyperFrames 或自研 WebGPU/WebCodecs 渲染层。
4. **便于灰度上线。** 早期只开放 `programmatic` 片段，成熟后再扩大到整片生成，不破坏剪映现有草稿协议。

### 4.4 分层架构图

```
┌──────────────────────────────────────────────────────────────────┐
│  用户交互层                                                        │
│  AI 对话面板 / 模板入口 / 素材选择 / 剪映时间线编辑器              │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌──────────────────────────────────────────────────────────────────┐
│  Agent 编排层                                                     │
│  Director -> Script -> Asset + Audio -> Editor -> Review          │
│  职责：意图拆解、分镜、素材/音频准备、DSL 生成、质量审查            │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌──────────────────────────────────────────────────────────────────┐
│  约束与状态层                                                     │
│  Creative Context / Timeline DSL / Component Registry / Schema    │
│  职责：统一上下文、组件白名单、字段校验、版本与回滚                │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌──────────────────────────────────────────────────────────────────┐
│  Harness 验证层                                                   │
│  Schema 校验 -> 素材校验 -> 规则校验 -> 首帧/关键帧 -> 视觉审查    │
│  职责：先拦截低成本错误，再进入昂贵渲染                            │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               v
┌──────────────────────────────────────────────────────────────────┐
│  渲染与编辑器集成层                                               │
│  剪映 WASM/WebCodecs 编辑器 + Remotion Player/Renderer + 导出队列 │
│  职责：可编辑草稿、程序化片段预览、客户端/服务端导出               │
└──────────────────────────────────────────────────────────────────┘
```

### 4.5 Agent 编排与状态流

多 Agent 的价值不是“角色更多”，而是把不同失败类型隔离到可局部重跑的步骤中。素材失败不应重跑脚本，音画不同步不应重跑素材，视觉溢出不应重跑整条 pipeline。

```
Director Agent
  输入：用户 prompt、平台、画幅、用户素材
  输出：CreativeBrief
        |
        v
Script Agent
  输出：Script + Storyboard（镜头、时长、旁白、scene_type）
        |
        +--------------------+
        |                    |
        v                    v
Asset Agent             Audio Agent
输出：MaterialList       输出：AudioAssets + timestamps
        |                    |
        +---------+----------+
                  v
Editor Agent
  输出：Timeline DSL + VideoSpec + Draft Patch
                  |
                  v
Review Agent + Harness
  输出：ReviewReport / FixPatch / RenderReady
```

| 设计点 | 为什么这样做 | 取舍 |
|---|---|---|
| Director 统一拆解任务 | 避免每个 Agent 自行理解用户意图导致风格漂移 | 多一次 LLM 调用，但换来上下文一致性 |
| Asset 与 Audio 并行 | 素材生成和 TTS 互不依赖，可缩短等待时间 | 需要最终由 Editor 对齐时长 |
| Editor 只生成 DSL/VideoSpec | 比直接生成 JSX 更稳定、更易验证 | 创意自由度低于任意代码 |
| Review 独立成 Agent | 把生成者和质检者分开，减少自证正确 | 增加视觉模型成本，但只抽关键帧 |

### 4.6 验证管线设计

验证顺序按“成本从低到高”排列：能在毫秒级发现的问题，不允许进入分钟级渲染。

```
Agent 输出 DSL / VideoSpec / Code
        |
        v
1. Schema 校验
   component 是否存在、props 是否合法、时间线是否连续
        |
        v
2. 资源校验
   素材 URL、字体、音频时长、授权元数据
        |
        v
3. 静态规则校验
   Remotion API 白名单、ESLint、确定性随机、CSS 限制
        |
        v
4. 快速预览
   首帧 + 关键帧截图，检查黑帧、溢出、遮挡
        |
        v
5. 完整渲染
   服务端 renderer / 客户端 web-renderer / Lambda
        |
        v
6. 渲染后审查
   时长、分辨率、fps、音画同步、视觉评分
```

| 层级 | 主要拦截 | 成本 | 失败处理 |
|---|---|---|---|
| Schema | 字段错、组件不存在、时间线重叠 | 极低 | 返回字段级错误给 Editor Agent |
| 资源 | 素材缺失、字体缺失、音频过长 | 低 | Asset/Audio Agent 局部重跑 |
| 静态规则 | API 误用、非确定性、CSS 不兼容 | 低 | Editor Agent 修复或降级路径 1 |
| 快速预览 | 黑帧、文字溢出、构图明显错误 | 中 | Review Agent 输出 FixPatch |
| 完整渲染 | Chrome/FFmpeg/WebCodecs/OOM | 高 | 降并发、降分辨率、切服务端兜底 |
| 渲染后审查 | 文件损坏、音画不同步、质量不达标 | 中 | 回到对应局部步骤 |

### 4.7 路径选择：为什么 MVP 主推 JSON -> 组件映射

| 方案 | 优点 | 风险 | 结论 |
|---|---|---|---|
| JSON -> 组件映射 | 安全、稳定、可验证、适合剪映草稿协议 | 创意自由度受组件库限制 | MVP 主路径 |
| LLM 直接生成 React 代码 | 表达力强，适合复杂动效 | 语法、安全、渲染稳定性都更难控 | P2 / 高级模式 |
| 直接生成 MP4 | 链路短，用户立即看到结果 | 不可编辑，不符合剪映产品心智 | 不作为主路径 |
| 全部走视频生成模型 | 真实画面能力强 | 成本高、不可控、难批量一致 | 只用于真实画面素材 |

### 4.8 与剪映现有架构的关系

Harness 不替代剪映的 WASM/WebCodecs 编辑引擎。正确关系是：剪映编辑器负责真实素材、多轨编辑、实时预览和用户交互；Harness + Remotion 负责生成程序化片段和可编辑草稿补丁。

```
┌──────────────────────────┐       ┌──────────────────────────┐
│ 剪映编辑器引擎            │       │ Harness + Remotion         │
│ WASM / WebCodecs          │       │ Agent / DSL / Renderer     │
├──────────────────────────┤       ├──────────────────────────┤
│ 真实视频裁剪、拼接、调色   │       │ 标题、图表、字幕动画、卡片   │
│ 多轨时间线交互             │       │ 程序化片段生成与质检         │
│ 本地预览与导出             │       │ 低成本批量生成               │
└────────────┬─────────────┘       └────────────┬─────────────┘
             │                                  │
             └────────── Draft / Timeline DSL ──┘
                              │
                              v
                       用户可编辑的视频草稿
```

---

<a id="section-5"></a>

## 五、Harness 核心模块详解

本章只展开 Harness 的核心模块机制；Agent 角色、端到端状态流和剪映落地任务不在这里重复，分别见第 4、6 部分。

### 5.1 代码生成管线

#### 两种生成路径

| 维度 | 路径 1：JSON -> 组件映射 | 路径 2：LLM 直接生成 React 代码 |
|---|---|---|
| 原理 | LLM 生成 VideoSpec / Timeline DSL，系统映射预制组件 | LLM 生成 JSX，经 Babel/ESLint/沙箱后执行 |
| 安全性 | 高，JSON 不可执行 | 中，需要 API 白名单、CSP、iframe/Worker 隔离 |
| 稳定性 | 高，字段可验证 | 中，容易出现语法和渲染错误 |
| 表达力 | 中，受组件库限制 | 高，可生成复杂动效 |
| MVP 决策 | 主路径 | P2 / 高级能力 |

路径 1 的执行链路：

```
Editor Agent 输出 VideoSpec
        |
        v
Zod / JSON Schema 校验
        |
        v
Component Registry 查找组件
        |
        v
VideoSpecRenderer 组装 Remotion Composition
        |
        v
Player 预览 / Renderer 导出
```

路径 2 只在组件库无法覆盖的高级场景启用，必须经过四道约束：移除 import/export、Babel 编译、ESLint 规则、白名单 API 注入。失败三次后回退路径 1。

### 5.2 组件库与注册表

组件库的目标是把 Agent 的创作空间限制在“足够好用但可验证”的范围内。MVP 不追求覆盖所有视觉效果，只覆盖信息型和模板化视频的高频结构。

| 组件 | 职责 | 核心 props |
|---|---|---|
| `AnimatedTitle` | 标题动画 | `text`, `fontSize`, `color`, `animation` |
| `SubtitleOverlay` | 字幕叠加 | `text`, `start`, `duration`, `style` |
| `KenBurnsImage` | 图片缩放平移 | `src`, `zoom`, `pan`, `duration` |
| `DataChart` | 数据图表 | `data[]`, `chartType`, `title`, `unit` |
| `ProgressBar` | 进度条 | `value`, `max`, `label`, `color` |
| `ImageSlideshow` | 图片轮播 | `images[]`, `transition`, `interval` |
| `EndCard` | 结尾卡片 | `title`, `subtitle`, `logo`, `cta` |
| `Transition` | 转场效果 | `type`, `duration` |
| `BackgroundGradient` | 动态背景 | `colors[]`, `animation`, `speed` |
| `LowerThird` | 下方字幕条 | `title`, `subtitle`, `style` |

注册表要同时服务三个对象：

```typescript
interface ComponentRegistryEntry {
  name: string;
  component: React.FC<any>;
  schema: z.ZodType<any>;
  description: string;
  category: 'title' | 'chart' | 'image' | 'transition' | 'background' | 'overlay';
  examples: Array<Record<string, unknown>>;
}
```

| 使用方 | 需要注册表提供什么 |
|---|---|
| Agent | 组件说明、适用场景、props schema、示例 |
| 校验器 | 组件是否存在、props 类型和范围是否合法 |
| 属性面板 | 根据 schema 自动生成可编辑控件 |

### 5.3 Timeline DSL / VideoSpec 契约

第 4 部分已经说明为什么 DSL 是核心中间层；这里给出最小契约。VideoSpec 是 DSL 面向 Remotion 的投影，不应承载全部剪映草稿信息。

```typescript
interface VideoSpec {
  composition: {
    width: number;
    height: number;
    fps: number;
    durationInFrames: number;
  };
  scenes: Scene[];
}

interface Scene {
  id: string;
  component: string;
  props: Record<string, unknown>;
  start: number;
  durationInFrames: number;
  transitions?: {
    in?: { type: string; duration: number };
    out?: { type: string; duration: number };
  };
}
```

关键约束：

1. `start + durationInFrames` 必须连续或有明确转场重叠规则。
2. `component` 必须来自注册表。
3. `props` 必须通过组件 schema。
4. 素材引用必须来自 `MaterialList` 或用户上传资产，不能由 Agent 编造 URL。
5. VideoSpec 可重新生成 Remotion 片段，但用户可编辑信息仍应保存在 Timeline DSL / 剪映草稿里。

### 5.4 质量保障模块

质量保障分三层：结构正确、渲染成功、视觉可用。

| 层级 | 检查内容 | 工程实现 |
|---|---|---|
| 渲染前验证 | 尺寸、fps、时长、素材、字体、时间线连续性 | `validateTimeline()` / `validateVideoSpec()` |
| 渲染中监控 | delayRender 超时、Chrome 崩溃、OOM、编码失败 | 渲染任务队列 + progress/error hooks |
| 渲染后审查 | 文件完整性、分辨率、fps、时长、音画同步、关键帧质量 | ffprobe + 抽帧 + Review Agent |

Review Agent 不替代硬规则。黑帧、尺寸、时长、素材缺失应由程序检测；构图、可读性、视觉层次再交给视觉模型辅助判断。

### 5.5 错误恢复与降级

```
失败发生
  |
  +-- Schema / props 错误 -> Editor Agent 修正 DSL / VideoSpec
  +-- 素材错误 -> Asset Agent 重取/重生成/转码
  +-- 音频错误 -> Audio Agent 重算时长与字幕时间戳
  +-- 渲染错误 -> 降并发 / 降分辨率 / 切服务端兜底
  +-- 视觉错误 -> Review Agent 输出 FixPatch，Editor Agent 局部修正
```

| 降级级别 | 触发条件 | 降级动作 |
|---|---|---|
| Level 1 | 代码生成 3 次失败 | 路径 2 回退路径 1 |
| Level 2 | 复杂动效渲染失败 | 替换为简单组件或静态关键帧 |
| Level 3 | 客户端渲染失败 | 切服务端 renderer |
| Level 4 | 完整渲染仍失败 | 生成可编辑草稿 + 标记失败片段，交给用户处理 |

### 5.6 性能模块

性能优化不应早于主链路稳定。MVP 先做低清预览和任务队列，P2 再做片段缓存、增量渲染和分布式。

| 模块 | MVP | P2/P3 |
|---|---|---|
| 预览分级 | Player 实时预览、低清预览、关键帧截图 | 根据设备动态选择预览质量 |
| 缓存 | Bundle 缓存、素材缓存 | 帧缓存、片段缓存、hash 级复用 |
| 并发 | 服务端队列限流 | Lambda / 自建分片渲染 |
| 客户端渲染 | POC `@remotion/web-renderer` | WebCodecs 导出、IndexedDB 缓存 |

### 5.7 素材与音频模块

| 模块 | 关键实现 | 验收标准 |
|---|---|---|
| 素材管理 | 统一 `assetId`、格式、尺寸、时长、来源、授权状态、CDN URL | 渲染环境可访问，授权状态可追踪 |
| 图片/视频转码 | 图片保留 PNG/WebP，视频统一 MP4/H.264 或剪映支持格式 | Remotion 和剪映编辑器都能加载 |
| TTS 对齐 | TTS 返回句级/词级时间戳，反向修正 scene duration | 字幕与配音误差小于 100ms |
| BGM 混音 | 音量曲线、口播 ducking、淡入淡出 | 背景音不压过人声 |

### 5.8 Harness 模块边界总结

| 模块 | 本章职责 | 与第 6 部分关系 |
|---|---|---|
| 代码生成管线 | 说明双路径和约束机制 | 第 6 部分拆成任务 4-7、16 |
| 组件注册表 | 说明组件、schema、catalog | 第 6 部分拆成任务 2-3 |
| DSL / VideoSpec | 说明 Remotion 投影契约 | 第 6 部分落到剪映 Draft Patch |
| 质量保障 | 说明验证与审查机制 | 第 6 部分拆成任务 9、11、12 |
| 性能优化 | 说明缓存、预览、并发策略 | 第 6 部分拆成任务 17、18、20、22 |
| 素材音频 | 说明素材和音频处理边界 | 第 6 部分拆成任务 13、14 |

---

<a id="section-6"></a>

## 六、剪映落地方案与实施计划

第 6 部分只回答“剪映团队怎么落地”：如何接入现有编辑器、怎样分阶段交付、每个开发任务怎么做。Remotion 原理见第 2 部分，Harness 架构理由见第 4 部分，核心模块机制见第 5 部分。

### 6.1 架构边界与许可证门槛

Remotion 是否进入剪映主链路，不只是技术问题，也是企业采购和产品边界问题。对 5000+ 人商业产品，架构评审必须先确认三件事：

1. **许可证门槛：** Remotion Company / Enterprise License、Automators 条款、客户端渲染计费方式必须在 Phase 0 明确；未明确前，Remotion 只能作为 POC 或可替换渲染目标。
2. **产品边界：** MVP 目标是“AI 生成可编辑草稿”，不是直接生成不可编辑 MP4，也不是替代剪映主编辑器。
3. **技术解耦：** Timeline DSL 是主协议，Remotion VideoSpec 只是渲染投影；如果许可证或 web-renderer 成熟度受阻，可以切 HyperFrames 或自研渲染器。

**MVP 非目标：**

| 非目标 | 原因 |
|---|---|
| 不替代 Seedance/Kling 生成真实画面 | Remotion 擅长程序化内容，不生成真实世界视频 |
| 不做全自动不可编辑成片 | 剪映的核心心智是编辑器，用户必须能改字幕、素材、节奏 |
| 不默认开放 LLM 直接生成 React 代码 | 安全和稳定性成本高，P2 再作为高级路径验证 |
| 不首期上线 Lambda / 分布式渲染 | MVP 先验证生成质量和编辑链路，规模化后再优化成本/速度 |
| 不承诺完整客户端导出 | `@remotion/web-renderer` 仍需 POC，高清导出先用服务端兜底 |

### 6.2 剪映侧集成架构

```
┌──────────────────────────────────────────────────────────────────┐
│ 剪映 Web 编辑器                                                   │
│ AI 对话面板 / 时间线 / 预览区 / 属性面板 / 版本历史               │
└──────────────────────────────┬───────────────────────────────────┘
                               │ Draft Patch / Timeline DSL
                               v
┌──────────────────────────────────────────────────────────────────┐
│ AI Video Pipeline Service                                        │
│ Orchestrator / Context Store / Component Registry / Validation   │
└───────────────┬───────────────────────────────┬──────────────────┘
                │                               │
                v                               v
┌──────────────────────────────┐   ┌──────────────────────────────┐
│ Asset & Audio Services        │   │ Render Services               │
│ 素材、TTS、BGM、授权、转码     │   │ Player、Renderer、WebCodecs、队列│
└───────────────┬──────────────┘   └───────────────┬──────────────┘
                │                                  │
                v                                  v
        剪映素材库 / CDN                    预览视频 / 关键帧 / MP4
```

前端不直接调用 LLM，也不直接拼 Remotion 代码。前端只提交用户意图和素材，接收 Pipeline 状态、Draft Patch、预览结果和可编辑参数。

### 6.3 前后端职责划分

| 层 | 职责 | 不做什么 |
|---|---|---|
| AI 对话面板 | 收集 prompt、画幅、平台、素材；展示步骤进度 | 不承载 Agent prompt 逻辑 |
| 时间线 Store | 应用 Draft Patch，维护用户可编辑状态 | 不直接理解 Remotion 组件内部实现 |
| 属性面板 | 根据组件 schema 展示可编辑参数 | 不写死每个组件表单 |
| Pipeline Orchestrator | 调度 Agent、状态机、重试、版本 | 不保存大文件本体 |
| Asset Service | 素材入库、转码、授权元数据、CDN URL | 不让 Agent 编造外部 URL |
| Render Service | 预览、导出、关键帧、渲染日志 | 不决定创意策略 |
| Validation Service | DSL、VideoSpec、素材、视觉检查 | 不生成内容 |

### 6.4 分阶段实施

| 阶段 | 时间 | 目标 | 退出标准 |
|---|---|---|---|
| Phase 0：技术预研 | 1 周 | 确认 Remotion 许可证、客户端渲染可行性、剪映草稿协议映射边界 | 法务/采购结论明确；跑通 1 个 Remotion 片段插入草稿 |
| Phase 1：DSL 与组件 MVP | 2 周 | 建立 Timeline DSL、组件注册表、10 个基础组件、JSON->组件映射 | 3 类模板视频可从 DSL 预览 |
| Phase 2：Agent Pipeline MVP | 2 周 | Director/Script/Editor Agent 串联，素材和音频先用模拟或已有能力 | 一句话生成 30 秒可编辑草稿，成功率 70%+ |
| Phase 3：Harness 验证与服务端渲染 | 2 周 | Schema、时间线、素材、首帧、渲染后校验；服务端导出 | 自动拦截黑帧/缺素材/时长错/文字溢出 |
| Phase 4：素材、音频与编辑器深度集成 | 2 周 | Asset/Audio Agent 接入，Draft Patch 应用到真实时间线 | 用户可编辑字幕、替换素材、调整智能片段参数 |
| Phase 5：客户端渲染与性能优化 | 3-4 周 | WebCodecs/web-renderer POC、缓存、低清预览、增量渲染 | 30 秒低清预览 < 10 秒；最终导出稳定 |
| Phase 6：规模化与商业化评估 | 2 周 | Lambda/队列/监控/成本模型/企业许可证落地 | 成本、成功率、延迟、采购路径可用于上线决策 |

### 6.5 开发任务清单

| # | 模块 | 难度 | 优先级 | MVP |
|---|---|---|---|---|
| 1 | 草稿协议 JSON ↔ Timeline DSL ↔ VideoSpec 转换器 | 中 | P0 | 是 |
| 2 | 核心组件库 v1（10 个组件） | 中 | P0 | 是 |
| 3 | 组件注册表 + Zod Schema + 属性面板 schema | 中 | P0 | 是 |
| 4 | JSON -> 组件映射渲染器 | 中 | P0 | 是 |
| 5 | Director/Script/Editor Agent 集成 | 高 | P0 | 是 |
| 6 | 代码沙箱 + Babel 编译（路径 2 内部 POC） | 高 | P2 | 否 |
| 7 | Remotion ESLint / 静态规则集成 | 低 | P1 | 是 |
| 8 | Remotion Player 预览集成 | 中 | P0 | 是 |
| 9 | 渲染前验证 | 中 | P0 | 是 |
| 10 | 服务端渲染管线 | 中 | P0 | 是 |
| 11 | 自动重试与局部修复 | 中 | P1 | 是 |
| 12 | Review Agent 质检集成 | 中 | P1 | 是 |
| 13 | 素材管理器 | 中 | P1 | 是 |
| 14 | 音频同步 | 中 | P1 | 是 |
| 15 | 进度追踪 + WebSocket/SSE | 中 | P1 | 是 |
| 16 | LLM 直接生成代码（路径 2） | 高 | P2 | 否 |
| 17 | 帧/片段缓存 | 高 | P2 | 否 |
| 18 | Lambda / 分片渲染 | 中 | P2 | 否 |
| 19 | 视觉回归测试 | 中 | P2 | 否 |
| 20 | 浏览器端渲染 | 高 | P2 | 否 |
| 21 | 多画幅适配 | 中 | P3 | 否 |
| 22 | 性能优化（预加载/低内存/增量渲染） | 高 | P3 | 否 |

### 6.6 任务 1-5：MVP 主链路怎么做

#### 任务 1：草稿协议 JSON ↔ Timeline DSL ↔ VideoSpec 转换器

- **目标：** 建立剪映草稿、Timeline DSL、Remotion VideoSpec 三者之间的转换层。
- **实现：** 定义最小 DSL，覆盖 `video/image/audio/subtitle/programmatic/effect` 六类轨道；实现 `draftToTimelineDsl()`、`timelineDslToDraftPatch()`、`timelineToVideoSpec()`；为每类 clip 建立字段映射表，标注可逆字段和只读字段。
- **验收：** 3 个样例草稿可往返转换；开始时间和时长误差小于 1 帧；转换函数有单元测试覆盖。
- **风险：** 剪映草稿协议版本变化频繁，需要 schema version 和迁移层。

#### 任务 2：核心组件库 v1

- **目标：** 提供 Agent 可组合的视频积木，避免 MVP 阶段自由生成代码。
- **实现：** 先做 10 个组件：标题、字幕、Ken Burns 图片、图表、进度条、轮播、结尾卡、转场、背景、LowerThird。所有动画用 `useCurrentFrame()`、`interpolate()`、`spring()`，不使用 CSS transition。
- **验收：** 每个组件支持 9:16/16:9；长中文、Logo 缺失、极端数据不溢出；有示例 props 和关键帧截图。
- **风险：** 组件过少限制表达，组件过多增加 Agent 选择难度；先覆盖高频场景，再根据真实日志扩展。

#### 任务 3：组件注册表 + Zod Schema

- **目标：** 让 Agent、校验器、属性面板共用同一套组件契约。
- **实现：** 每个组件定义 schema、描述、示例、默认值、适用场景；`toCatalog()` 生成 Agent 可读 catalog；属性面板根据 schema 自动生成表单。
- **验收：** Agent 只能引用注册表组件；错误能精确到 `scene_id/component/field`；用户能在属性面板修改智能片段参数。
- **风险：** catalog 太长会增加 token 成本，建议默认注入摘要，需要时按组件展开。

#### 任务 4：JSON -> 组件映射渲染器

- **目标：** 实现 MVP 主路径，从 VideoSpec 稳定生成 Remotion Composition。
- **实现：** `VideoSpecRenderer` 遍历 scenes，按 `start/durationInFrames` 包装为 `Sequence`；从注册表取组件；转场通过白名单处理；非法组件或 props 在渲染前失败。
- **验收：** 时间线连续；同一 VideoSpec 多次渲染一致；组件不存在、props 不合法、转场不支持时返回结构化错误。
- **风险：** 转场会影响相邻 scene 的帧数，需要在 VideoSpec 生成阶段提前扣除转场帧。

#### 任务 5：Director/Script/Editor Agent 集成

- **目标：** 让自然语言变成可验证的 CreativeBrief、Storyboard 和 VideoSpec。
- **实现：** Agent 输出全部使用 JSON schema；每次调用保存 prompt、model、输入、输出、耗时和版本；素材 URL 只能来自 MaterialList。
- **验收：** 20 条内部测试 prompt 中至少 14 条生成可预览草稿；失败可归类；不编造不存在的素材 URL 或组件名。
- **风险：** 直接生成最终代码不稳定，MVP 只生成 JSON/DSL。

### 6.7 任务 6-15：稳定性与编辑器集成

| 任务 | 怎么做 | 验收重点 | 主要风险 |
|---|---|---|---|
| 6 代码沙箱 + Babel | 仅做内部 POC，AST 解析、白名单 API、iframe/Worker + CSP | 验证路径 2 可行性，不进入 MVP 主链路 | `new Function()` 生产安全风险 |
| 7 静态规则 | 对路径 1 做 DSL/素材/时间线规则，对路径 2 POC 接入 `@remotion/eslint-plugin` | 禁止素材编造、时间线错误、非确定性渲染 | 规则过重会拖慢主链路 |
| 8 Player 预览 | 预览区接入 Remotion Player，以 frame number 同步主时间线 | 参数修改 1 秒内刷新 | 双播放器时间偏移 |
| 9 渲染前验证 | 检查尺寸、fps、scene 连续、素材、字体、音频时长 | 缺素材/时长错/字体缺失可拦截 | 远程素材 HEAD 不可靠 |
| 10 服务端渲染 | `bundle()` + `selectComposition()` + `renderMedia()`，队列管理并发 | 30 秒 1080p 稳定导出 | Chrome/FFmpeg/字体环境不一致 |
| 11 自动修复 | 错误分类后路由到对应 Agent 局部修复 | 常见 schema/素材错误自动修复率 80%+ | Agent 越修越偏 |
| 12 Review Agent | 抽 0/25/50/75/99% 关键帧做视觉审查 | 文字溢出、黑帧、遮挡能识别 | 视觉模型评分波动 |
| 13 素材管理器 | 统一 assetId、格式、来源、授权、CDN URL | 渲染环境可访问，授权可追踪 | 公网素材版权风险 |
| 14 音频同步 | TTS 时间戳反向修正 scene duration，BGM ducking | 字幕与配音误差 <100ms | 先定画面再配音会失步 |
| 15 进度追踪 | WebSocket/SSE 推送状态，支持取消/恢复/局部重跑 | 刷新页面可恢复 run 状态 | 暴露底层错误给用户 |

### 6.8 任务 16-22：P2/P3 能力

| 任务 | 上线前提 | 价值 |
|---|---|---|
| 16 LLM 直接生成代码 | 沙箱、ESLint、Review Agent 稳定 | 支持复杂定制动画 |
| 17 帧/片段缓存 | VideoSpec hash 和组件版本稳定 | 降低重复预览和导出成本 |
| 18 Lambda / 分片渲染 | 服务端渲染链路稳定且成本可归因 | 支持长视频和批量生成 |
| 19 视觉回归测试 | 组件库进入稳定维护期 | 防止组件改动破坏模板 |
| 20 浏览器端渲染 | `@remotion/web-renderer` POC 通过，许可证明确 | 对齐 Web-first，降低云渲染成本 |
| 21 多画幅适配 | 组件布局系统稳定 | 一份脚本适配多平台 |
| 22 性能优化 | 有真实瓶颈数据 | 提升预览响应和低内存稳定性 |

### 6.9 MVP 验收标准

| 维度 | 指标 |
|---|---|
| 生成成功率 | 内部 50 条标准 prompt 中，70%+ 生成可预览草稿 |
| 可编辑性 | 字幕、图片、音频、程序化片段参数均可在剪映编辑器中修改 |
| 预览速度 | 30 秒视频低清预览首帧 < 5 秒，完整低清预览 < 10 秒 |
| 导出稳定性 | 30 秒 1080p 服务端导出成功率 95%+ |
| 质量拦截 | 黑帧、缺素材、文字溢出、时长不一致可自动拦截 |
| 成本可观测 | 每次 run 记录 LLM、素材生成、TTS、渲染资源成本 |
| 合规 | Remotion 许可证、素材版权、字体授权、BGM 授权均可追踪 |

**50 条标准 prompt 数据集建议：**

| 类别 | 数量 | 目的 |
|---|---:|---|
| 知识科普 / 概念解释 | 10 | 验证文字结构、字幕、标题动画和信息密度 |
| 产品功能介绍 / SaaS 宣传 | 10 | 验证品牌卡片、Logo、CTA、截图素材组合 |
| 数据报告 / 榜单盘点 | 10 | 验证图表、数字动画、进度条、数据可读性 |
| 营销模板 / 活动预告 | 10 | 验证模板复用、色彩风格、转场和节奏 |
| 图文混剪 / 用户素材改编 | 10 | 验证用户素材接入、字幕同步、可编辑草稿质量 |

每条 prompt 都应记录目标画幅、目标时长、素材输入、期望输出结构和人工验收标签；否则“70% 生成成功率”不可复现，也无法比较不同 Agent / 模型版本。

### 6.10 商业与许可证注意事项

1. **Remotion 许可证必须前置确认。** 对 5000+ 人商业产品，不能按个人或小团队开源使用理解，应按 Company / Enterprise License 与 Remotion 官方确认：开发者席位、Automators 计费、客户端渲染是否计费、嵌入剪映产品是否需要额外条款。
2. **保留替代渲染路线。** Timeline DSL 必须与 Remotion 解耦，避免采购或许可证谈判失败后推倒重来。HyperFrames 或自研 WebGPU/WebCodecs 程序化渲染层可以复用同一 DSL。
3. **素材版权要进入系统设计。** Agent 不应默认从公网抓取素材；素材来源、商用授权、生成模型条款、字体授权、BGM 授权都要成为 Asset metadata。
4. **MVP 不建议追求全自动成片。** 更稳的产品形态是“AI 生成可编辑草稿 + 用户快速微调 + 一键导出”。
5. **Remotion 的直接价值不是替代视频生成模型。** 它主要降低信息型/模板化视频成本；真实画面场景仍应使用 Seedance/Kling 等模型，再由剪映编辑器完成二次创作与合成。

---

<a id="section-7"></a>

## 七、成本与商业分析

### 7.1 渲染成本对比

| 方案 | 单条 30 秒 1080p | 1000 条/月 | 年费 |
|---|---|---|---|
| **Remotion 客户端渲染** | **~¥0.05**（仅 LLM 代码生成） | **~¥50** | **~¥600** |
| Remotion Lambda 渲染 | ~¥2-3.5 | ~¥2,000-3,500 | ~¥24K-42K |
| Seedance 2.0 | ~¥23+ | ~¥23,000+ | ~¥276K+ |
| Kling | ~¥12-30 | ~¥12,000-30,000 | ~¥144K-360K |

### 7.2 Remotion 许可证费用

| 许可证类型 | 费用 | 适用 |
|---|---|---|
| Free | ¥0 | 个人 / ≤3 人公司 |
| Company License | $25/开发者席位/月，最低 $100/月 | ≥4 人公司 |
| Remotion for Automators | $0.01/次渲染，最低 $100/月 | 构建 SaaS / 自动化视频工具 |
| Enterprise License | $500/月起 | 大型企业（定制条款、SLA、专属支持） |

**对剪映（5000+ 人公司）的影响：**
- 需购买 Enterprise License（$500/月起）
- 如做自动化渲染服务，可能触发 Automators 条款（$0.01/次渲染）
- **纯客户端渲染是否算 Automators 渲染需与 Remotion 团队确认**
- 联系方式：hi@remotion.dev / [官网预约通话](https://www.remotion.dev/contact)

### 7.3 投资回报分析

| 维度 | 数值 |
|---|---|
| Remotion 年费（Enterprise） | ~¥43K-86K |
| 节省的视频生成模型成本（1000 条/月） | ~¥276K+/年 |
| **净节省** | **~¥190K-233K/年** |
| 开发投入（15 个 MVP 模块） | 6-8 周 × 3-4 开发者 |

**结论：** 即使按保守估计（1000 条/月），Remotion 方案在投入运营后 1-2 个月内即可收回 License + 开发成本。

---

<a id="section-8"></a>

## 八、风险评估与应对策略

### 8.1 技术风险

| 风险 | 严重度 | 应对策略 |
|---|---|---|
| 客户端渲染 CSS 子集限制 | 中 | 限制 MVP 组件的 CSS 使用范围；启用 HTML-in-canvas 实验 |
| AI 生成代码质量不稳定 | 中 | 路径 1（JSON 映射）优先；3 次失败降级；ESLint 拦截 |
| Remotion Lambda 成本随用量增长 | 低 | 客户端渲染优先；Lambda 仅作 fallback |
| 渲染速度慢（单机 1-3 fps） | 中 | Lambda 分布式；帧缓存 + 增量渲染 |
| 用户设备性能不足 | 低 | 检测设备能力；性能不足提示云端渲染 |

### 8.2 商业风险

| 风险 | 严重度 | 应对策略 |
|---|---|---|
| Remotion 许可证成本上升 | 低 | 当前定价稳定；锁定多年合同；评估 HyperFrames（Apache 2.0）作为替代 |
| Automators 条款适用范围不确定 | 中 | 提前与 Remotion 团队沟通确认 |
| Remotion 团队维护中断 | 低 | Remotion AG 全职维护，社区活跃；源码可审计 |

### 8.3 竞争风险

| 风险 | 严重度 | 应对策略 |
|---|---|---|
| HyperFrames（Apache 2.0）快速成熟 | - | 作为技术储备持续关注；必要时切换 |
| 视频生成模型成本大幅下降 | 低 | Remotion 优势在可控性和精确性，不仅是成本 |
| 竞品率先实现类似方案 | 中 | 加快 MVP 落地；组件库和 Harness 是壁垒 |

### 8.4 替代方案储备

| 方案 | 许可证 | 优势 | 劣势 |
|---|---|---|---|
| **HyperFrames**（HeyGen 开源） | Apache 2.0 | 完全免费、AI 生成 HTML 更可靠 | 不支持浏览器端渲染、开源仅 4 个月 |
| **Motion Canvas** | MIT | 完全免费 | 不支持 HTML/CSS、无 AI 官方支持 |
| **自研渲染引擎** | 自有 | 完全可控、零许可证费 | 开发量大（3-6 个月） |

**建议：** MVP 阶段用 Remotion 快速验证，同步评估 HyperFrames 成熟度，中长期视情况决定是否切换或自研。

---

<a id="section-9"></a>

## 九、附录：参考来源

### Remotion 官方文档
- [Remotion 官网](https://www.remotion.dev/)
- [License & Pricing](https://www.remotion.dev/docs/license)
- [客户端渲染概览](https://www.remotion.dev/docs/client-side-rendering/)
- [@remotion/web-renderer](https://www.remotion.dev/docs/web-renderer/)
- [Generate Remotion Code using LLMs](https://www.remotion.dev/docs/ai/generate)
- [Dynamic Compilation](https://www.remotion.dev/docs/ai/dynamic-compilation)
- [Agent Skills](https://www.remotion.dev/docs/ai/skills)
- [Prompt to Motion Graphics 模板](https://github.com/remotion-dev/template-prompt-to-motion-graphics-saas)
- [@remotion/eslint-plugin](https://github.com/remotion-dev/remotion/blob/main/packages/eslint-plugin/src/index.ts)
- [@remotion/lambda](https://www.remotion.dev/docs/lambda)

### 社区实践
- [Harness Engineering for Video Agents — Starti.ai](https://starti.ai/news/harness-engineering-for-startiai-video-agents)
- [Making Videos with Claude Code and Remotion](https://blog.openreplay.com/making-videos-claude-code-remotion/)
- [Remotion Agent Skills Guide 2026](https://aividpipeline.com/blog/remotion-agent-skills-guide-2026)
- [AI-Assisted Video IDE Workflow](https://neural-nexus.net/build-an-ai-assisted-video-ide-workflow-with-remotion-generate-react-video-components-with-coding-agents-and-preview-in-remotion-studio/)
- [Remotion Best Practices: 30 Rules](https://www.ailinklab.com/en/skills/skill-remotion-best-practices/)

### Remotion 商用联系
- 邮箱：hi@remotion.dev
- 预约通话：https://www.remotion.dev/contact
- Enterprise 专线：通过 hi@remotion.dev 邮件联系

### 替代方案
- [HyperFrames GitHub](https://github.com/heygen-com/hyperframes)（Apache 2.0）
- [Motion Canvas GitHub](https://github.com/motion-canvas/motion-canvas)（MIT）

---

*本文档基于 Remotion 官方文档（2026 年 7 月最新版）、GitHub 仓库源码与 Issue、社区开源项目实践、剪映技术调研等多源信息交叉整理，供剪映团队评审参考。*
