---
title: "如何构建团队真正愿意用的软件工厂｜中文译述"
date: 2026-11-25T00:00:00+08:00
draft: false
taxonomies:
  tags: ["AI 研发效能", "软件工厂", "AI 智能体", "Agent Skills", "代码审查"]
extra:
  source: https://x.com/i/article/2125313266192887808
  hostname: x.com
  author: Ben Holmes
  original_title: "How to build a software factory you'll actually use"
  original_lang: en
---

## 与 AI 研发效能的相关性

**高度相关。** 这篇文章没有停在「把 issue 丢给 agent」这一步，而是给出一条渐进路线：从最朴素的 backlog 自动化出发，逐层加上人工判断（triage 与规格撰写）、让 agent 自己用 computer use 验证、再给 PR 做合并风险评估，最后得到一个团队愿意认真 review 其 PR 的软件工厂。对 AI 研发效能团队来说，它的可迁移之处在于把「结构化工作流 + 人工闸门 + 验证」当成可编码的工程产物：YAML blueprint、Markdown skill、PR 里的 visual proof 和风险评级，都能搬到任意 agent 栈里。文中具体数字（例如单个小修复的成本低于一美元）来自 Warp 的内部示例和一个演示应用，迁移到别的组织必须重新测量。

**出处**　X（Twitter）长文，作者 Ben Holmes（Warp），2026 年 11 月 24 日。本文为全文级中文译述，覆盖原文全部章节、论点、数据、代码与图表，非逐句对照翻译；封面与图 1–7 引用原文图片与视频封面，图内保留英文。

[How to build a software factory you'll actually use](https://x.com/i/article/2125313266192887808) · [作者发布推文](https://x.com/bholmesdev/status/2125351601431089418)

![封面｜软件工厂的构成示意](https://pbs.twimg.com/media/HX7BKEsXYAAPOjd.jpg)

## 引言

软件工厂有无数种定义。下面是 Ben Holmes 团队自己用的那一个：

> 云端的一条自动化循环：编排 agent，让软件随着时间越建越好。

很多团队把注意力放在「自动化循环」这一半，急着把一堆 issue 灌给 agent 来拉高速度。从那里起步很容易……但不少团队最后得到的是一个自己不敢信任的黑箱，或者一台「slop cannon（垃圾制造机）」——不停产出没人读的 PR。

所以作者认为，重点应该放在定义的另外两半：「在云端」和「随时间变得更好」：

- **在云端**：把「如何让 agent 写出更好的代码」这类知识集中起来，全团队共享。
- **随时间变得更好**：把这些知识系统化成一套分步工作流，让整个团队（以及 agent）都能贡献、度量并改进它。

文章会沿着这条演进路线走一遍：从普通的 backlog 自动化开始，演示怎么把它塑造成一个你愿意去 review 其 PR 的软件工厂。

覆盖的内容包括：

- 如何把一个基础的实现流水线写成一个 blueprint，然后接上 backlog 看它的表现
- 在哪里把开发者拉进来，用 triage 和规格撰写来判断产品需求
- 如何给 agent 一台「电脑」，让它验证自己的工作
- 如何评估合并风险，让你一眼看出哪些 PR 需要最仔细的 review

这套流程可以带到任何 agent 配置里。想跟着做，可以看他们软件工厂课程的 [Unit 2](https://www.warp.dev/learn/build-a-self-improving-software-factory/build-your-factory)，亲手搭一个工厂。

## 定义工厂

要搭 backlog agent，先列出 agent 应当执行的步骤序列。最直接的做法是把它写成一个 Markdown skill：

```markdown
1. Take in the user's request, including any links and attachments.
2. Implement the change in the repository and run static checks.
3. Open a draft pull request and send the user a link with a summary.
```

这已经足够简单。如果再给任务板应用（GitHub、Linear、Jira 等）挂一个 MCP 服务，我们甚至可以在自己工作的地方直接 review 产出。

但这还不是最确定性的写法。如果想在每一步强制输入输出的结构呢？想把不同步骤分派到不同的 agent 沙箱以管理上下文窗口呢？或者想把这些步骤编译成好看的图或状态机呢？

为了拿到这种灵活性，把步骤翻译成一份 YAML 模板。下面是他们采用的一种表示，遵循 [Warp Factories](https://warp.dev/factories) 的「blueprint」格式：

```yaml
blueprint:
  worker: foreman
  start:
    to: implement
  stages:
    implement:
      worker: builder
      inputs: |-
        request: the requester's ask in its exact words, with links and attachments.
      outputs: |-
        pr: the pull request that was opened or updated.
      prompt:
        file: stages/implement.md
      to: handoff
    handoff:
      worker: foreman
      inputs: |-
        report: the builder's result, summary, and links.
      prompt: Deliver the PR to the requester.
      to: end
```

逐项拆解这些属性：

**prompt** 告诉 agent 在这个阶段做什么。实现阶段的 prompt 单独放在一个 Markdown 文件里，方便边做边改。此刻它只要求 agent 实现改动并跑静态检查。**to** 字段指明工作接下来去哪儿。

**inputs** 描述一个阶段期望收到什么。这里的实现阶段会拿到一个 request，包含用户的原始请求以及其中的链接和附件。**outputs** 描述它应当交回什么：在这个例子里是一个 `pr`。

最后，**workers** 是彼此独立的沙箱，你可以把某个阶段的工作指派给它。这个例子用了两个，各自拥有独立的上下文窗口：一个负责与用户交互的 foreman，一个负责写代码的 builder。于是我们有了一个「前门 agent」，可以随时向它询问进展，而后端 agent 不受打扰地干活。

这样写工作流有两个好处。第一，输入和输出都有名字和结构，所以每次运行都被要求返回同样形状的结果。

第二，这种格式可以被可视化，随着工作流经系统而呈现。Warp 围绕这个格式做了可视化工具，能跟着阶段走、看到当前是哪个 worker 在处理：

![图 1｜把 blueprint 可视化为工作流，能看到每个阶段由哪个 worker 执行（原文图注：一个更复杂软件工厂的预览，包含 triage、写规格与 review 等额外步骤）](https://pbs.twimg.com/media/HX6iE_QW0AEe8Z8.jpg)

这也让我们可以顺手生成调试工具，弄清哪个 agent 在流水线的哪一步干活。

下面是他们内部调试工具的预览，用来追踪这条 2-agent 流水线里「foreman」和「builder」两个 agent 做的工作：

![图 2｜按 issue 追踪运行：每个打了标签的 issue 一次运行，带 issue 链接与产出的 PR 链接](https://pbs.twimg.com/media/HX6iIYMWUAAohCh.jpg)

每个打了标签的 issue 一次运行，带 issue 链接与产出的 PR 链接，是课程示例。

## 测试代码质量

工厂定义好了，现在送一些真实工作进去，用一份 issue backlog 试试。

首先需要一个把工作送进工厂的触发器。下面是一份 Warp Factories 的示例定义：只要 GitHub issue 被打上 `factory` 标签，就启动一次工厂任务。

```markdown
---
triggers:
  - provider: github
    event: issue_labeled
    filter:
      repos: [YOUR_GITHUB_ORG/my-markdown-app]
      labels: [factory]
      skip_bots: self
---
Use the issue that triggered this run as the request:

- author: the requester
- URL: the issue this work tracks; link the pull request on it
- title / body: the description to follow
```

把十个 issue 的 backlog 送进工厂后，结果是这样：

![图 3｜十个 issue 送进工厂后的产出](https://pbs.twimg.com/media/HX6iPMgWsAEj3-s.jpg)

对小型、定义明确的修复，这套东西相当好用。以这个[原生桌面应用](https://github.com/warpdotdev-demos/warp-learn-markdown-app)为例，用开放权重模型做一次小改动，每个 issue 的成本不到一美元。

但对更深入的功能开发，这些 PR 值得再看一眼。如果你只丢一句「去把它建出来」，agent 会自己「vibe」出哪些产品决策？

## 用 triage 和规格引入人工判断

看这个应用收到的 issue：

> 在文件侧栏里，加一个一次性展开或折叠所有文件夹的方式。

问题陈述本身很清楚，但如果这个 issue 落到我桌上，我大概会先问几个问题：

- 我们到底要不要在产品里加这个新能力？
- 如果要，它该放哪儿？侧栏里的新按钮、文件夹的右键菜单、macOS 菜单栏的 View 菜单，还是别的地方？

问题在于，我们的 backlog agent 无论需求多模糊，都急着写代码。把这条 issue 送过去后，我们收到的 PR 是这样的：

![图 4｜把模糊需求直接交给 backlog agent 后收到的 PR](https://pbs.twimg.com/media/HX6iTShW8AAEAe_.png)

嗯，它能跑。但个人而言，我更想要一个单一的切换按钮，而不是分开的两个控件；或者把它放到更安静的位置，比如应用的菜单栏。

而且，一个需求留下的开放问题越多，agent 出错的概率就越大。slop cannon 就是这么来的：agent 在没有人工判断的情况下不断吐出改动。我们需要一种方式，在实现 agent 撒野之前把它「卡住」。

为此，在实现阶段前面加两个阶段：**triage（分诊）**和**规格撰写（spec writing）**。

**Triage 充当路由器。** 它研究会请求，然后在三条路里选一条：

- 这是一个定义明确、可以直接进入实现的改动
- 这是一个存在开放产品决策、值得开一次协作会的功能
- 这只是一个需要回答的问题，根本不需要实现

**如果选中第二条**，就进入 spec 阶段。它给 agent 留出空间，先向开发者提问、就「该实现什么」达成一致；然后 agent 写一份规格，描述产品的预期行为以及实现要采用的技术方案。整条流程可以编码进前面的 blueprint：

```yaml
blueprint:
  worker: foreman
  start:
    to: triage
  stages:
    triage:
      worker: builder
      inputs: |-
        request: the requester's ask in its exact words, with links and attachments.
      outputs: |-
        goal: answer, change, or unclear.
        issue: the issue URL, or the decision that the work needs no issue.
        complexity: the assessed size, and any behavior choices left open.
        spec: whether a spec is needed, and why.
      prompt:
        file: stages/triage.md
      to:
        - stage: spec
          when: A spec is needed, or the requester asked for one.
        - stage: implement
          when: The goal is a change, and no spec is needed.
        - stage: handoff
          when: Otherwise. The request was answer-only, or is unclear and waiting on the requester.
    spec:
      worker: builder
      inputs: |-
        request: the requester's ask in its exact words, with links and attachments.
      outputs: |-
        spec_pr: the draft pull request holding the committed spec.
        decisions: the choices the requester made, and the assumptions recorded.
      prompt:
        file: stages/spec.md
      to:
        - stage: implement
          when: The requester approved the spec.
        - stage: spec
          when: The requester gave feedback on the spec. Revise it on the same branch and pull request.
    implement:
      worker: builder
      inputs: |-
        request: the requester's ask in its exact words, with links and attachments.
        triage: the triage outputs, with the request they came from.
        spec: the approved spec pull request, when there is one.
      outputs: |-
        pr: the pull request that was opened or updated.
      prompt:
        file: stages/implement.md
      to: handoff
    handoff:
      worker: foreman
      inputs: |-
        report: the worker's result, summary, and links.
      prompt: Deliver the result to the requester, including the PR link when there is one.
      to: end
```

这里，triage 用 `to` 字段列出供 agent 抉择的几条路径；而 `implement` 现在期望拿到 triage 的结果，以及可选的规格文档作为输入。

这份规格会成为构建时的上下文，其中「验证」一节则变成事后验收工作的清单。他们的倾向性做法是把规格作为代码提交，但放在内存里也可以。示例的 triage 与 spec prompt 见 Warp 学习课程里的 [「Adding stages for triage and speccing」](https://www.warp.dev/learn/build-a-self-improving-software-factory/tackle-larger-features-with-specs#adding-stages-for-triage-and-speccing)。

把「文件夹展开/折叠」这条请求走一遍这个流程后，agent 回来时带的是问题，而不是立刻倾泻代码：

- **Surface（入口）**：「expand all」和「collapse all」应该放在哪里？
- **Scope（范围）**：这个操作应该作用于侧栏里的所有文件夹，还是只作用于选中的文件夹及其子文件夹？
- **Open file（打开的文件）**：执行「collapse all」之后，当前打开文件所在的分支会怎样？

最后一个问题其实是我之前没想到的边界情况。如果你用过 Matt Pocock 的 [/grill-me skill](https://www.aihero.dev/skills-grill-me) 足够多次，大概也经历过这种时刻！而现在，agent 只在该拷问我们的时候才拷问。

在这个例子里，我回答：放进 View 菜单、作用于所有文件夹、保持打开文件所在文件夹为展开状态。结果如下：

![图 5｜按已批准的规格实现后的结果：macOS 的 View 菜单里出现 Expand All Folders 与 Collapse All Folders](https://pbs.twimg.com/media/HX6ibmGXkAABkuH.jpg)

实现现在遵循已批准的规格：macOS 的 View 菜单里有 Expand All Folders 和 Collapse All Folders。这是课程示例。

## 给 agent 一台电脑来验证工作

我们已经在实现之前把人工判断引入了流程。现在说另一半：在开 PR 之前的验证。

要拿到上面那张结果截图，我得把分支拉到本地、把应用端到端跑起来、想清楚要点击哪些用户流程，才能相信这个 PR 真的按描述工作。

当然，在本地 checkout 一个改动没什么问题，在 AI 之前的年代 review 同事的改动时甚至是理所当然的。但问问自己：你愿意拉下一个作者本人都没在自己机器上测过的 PR 吗？大概不愿意。那我们为什么能接受 agent 这么做？

所以，教 agent 如何在电脑上测试。这涉及 computer use：让 agent 模拟一个环境，把它做出来的东西端到端点一遍。今天已经有不少 agent harness 支持 computer use，而 Warp Factories 把它直接内置在 worker 的沙箱里（见[文档](https://docs.warp.dev/agents/capabilities/computer-use/)）。

具体到应用的验证指令，会写成一个 [skill](https://www.warp.dev/learn/build-a-self-improving-software-factory/add-verification-and-merge-risk#teach-workers-to-verify-their-work)。对这个 Electron 应用，他们的 UI 验证 skill 描述了如何启动应用、连接 DevTools、走完要求的流程，并录下证据：

```markdown
---
name: ui-verification
description: Prove a user-facing change works by driving the running app with computer use, and add a Visual proof section to the pull request description.
---
# UI verification

Use this whenever a change affects something a user can see or click.

## Start the app

Run `MARKY_DESKTOP_ENABLE_CDP=1 pnpm dev:desktop`.
Read the terminal output for the playground path and DevTools endpoint.
Use the auto-opened editable playground for verification.

## Walk the path

Walk the exact path the request describes. For a bug, show the symptom
is gone. For a feature, show the new behavior and its key interactions.
Use computer use to see and record the app. Prefer the Chrome DevTools
Protocol endpoint for interaction-heavy steps.

## Record the proof

Record a video of the flow. Add a screenshot for a final state worth
inspecting closely. Add a Visual proof section to the PR describing
the path you walked and what each recording or screenshot shows.
If the app cannot launch or the path cannot be reached, say so.

## Gotchas

- Screenshots are DPR 2: CSS pixels times 2.
- Right clicks need real Input.dispatchMouseEvent events; synthetic
  contextmenu events do not open base-ui context menus.
```

注：其中的「gotchas」是 agent 长期测试这个 skill 时积累下来的有用知识。作者在 AI Engineer 的[关于自我改进的演讲](https://www.youtube.com/watch?v=pz74r-4dGfQ)里更深入地讲了如何训练 agent 记录这类知识。

现在，agent 手里有一个 skill，可以在开 PR 之前验证自己的工作。下面是它拿到这个 skill 后，为一处窄屏下的侧滑侧栏改动所添加的示例「visual proof」段落：

![图 6｜附在 PR 里的 visual proof：agent 走过的路径、一段录屏，以及改动前后的截图](https://pbs.twimg.com/media/HX6j57fWEAA3FjH.jpg)

![图 7｜生成的演示录屏：展示侧滑菜单的行为；Warp Factories 会自动剪掉静止片段，并在录屏里高亮鼠标点击](https://pbs.twimg.com/amplify_video_thumb/2125316974808801280/img/XwYiJGcqXmEBjcKc.jpg)

图 7 为视频，[点此观看 MP4](https://video.twimg.com/amplify_video/2125316974808801280/vid/avc1/558x720/ZVsY7yIPQwZDHhXs.mp4?tag=29)。

## 评估合并风险

验证到位之后，再看一眼那一整墙已完成的工厂运行：

收件箱里堆着不少 PR 要 review；很难分辨哪些高风险、哪些还没做完、哪些只要扫一眼描述就能合。我需要一种办法，不用点进每一个 diff 就能判断。

这就是合并风险评估的用武之地。他们把方法做成一个 skill，可以在 [warp.dev/merge-risk](https://www.warp.dev/merge-risk) 下载试用。它让 agent 从五个类别评估自己的工作：

- **Alignment（对齐度）**：代码是否与 PR 描述和 issue 吻合，还是 worker 夹带了没写进文档的顺手改动？
- **Impact（影响面）**：爆炸半径有多大，比如是一个孤立的视图还是一个共享组件库；是否触及鉴权、权限、数据库 schema 等敏感区域？
- **Verification（验证）**：行为是否通过 computer use、端到端测试和单元测试验证过？
- **Rollback cost（回滚成本）**：这是一扇双向门（容易 revert 的提交、feature flag），还是一扇单向门（破坏性的 schema 变更、损坏的持久化状态）？
- **Error detection（错误可观测性）**：如果生产环境出问题，会报到监控或日志，还是错误被悄悄吞掉？

这个 skill 会把这些判断合成一个低、中、高风险分数，并点出推高风险的具体缺口。你可以把这些评估输出成 GitHub 标签、PR 描述条目，或者一整份 GitHub 评论报告。

还有一个值得做的动作：让实现者自己在开 PR 之前先跑这个 skill。这样它有机会提前缓解风险，比如补上缺失的错误日志、收集验证证据，再把评估结果放进交接里。他们把这个要求加进了自己的实现 prompt：

```markdown
Before opening a PR, use skills/merge-risk/SKILL.md to mitigate risk
ahead of PR, if possible. Report the "Impact" and "Rollback cost" as
sectioned headings in the PR, with a **Verdict** matching the risk
for that category, and a brief summary for each.
```

现在，PR 描述会在实现摘要之外带上 visual proof、影响面和回滚成本。下面是[一个格式良好的 PR 示例](https://github.com/bholmesdev/hubble.md/pull/322)，覆盖了验证、影响面和回滚成本。

## 继续建设工厂

希望这篇文章给了你一些思路，去搭一个你愿意信任、并开始往外推 PR 的软件工厂。只要覆盖了三件事——a) 结构化工作流、b) 人工判断、c) 验证——你就有了需要的全部工具。

从这里还能做很多。他们团队已经开始给开 PR 之前的代码审查做多版本基准测试，引入一个拥有独立上下文窗口的研究型 agent 来提升 token 效率，还在为 macOS 增加定制 runner 来测试原生 iOS 应用。

如果这些让你感兴趣，可以看看 [Warp Factories](https://www.warp.dev/factories)，并跟着他们的 [Learn 课程](https://www.warp.dev/learn/build-a-self-improving-software-factory/build-your-factory)亲手搭这些工作流。课程比这篇指南更深入，带完整的代码片段和示例，你可以用来搭自己的工厂。
