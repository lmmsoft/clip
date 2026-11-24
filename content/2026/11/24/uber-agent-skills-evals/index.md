---
title: "Uber 如何每月评估百万次 Agent Skill 调用｜中文译述"
date: 2026-11-24T00:00:00+08:00
draft: false
taxonomies:
  tags: ["AI 研发效能", "Agent Skills", "AI 智能体", "Evals"]
extra:
  source: https://www.uber.com/us/en/blog/optimizing-ubers-agent-skills/
  hostname: www.uber.com
  author: Andrew Mager、Jason Varbedain、Uday Kiran Medisetty
  original_title: "How Uber Evals 1 Million Agent Skill Invocations a Month"
  original_lang: en
---

## 与 AI 研发效能的相关性

**高度相关。** 这是上一篇《Uber 如何高效运营软件工厂》的续篇：前文讲怎么把智能体的单位成本降下来，本文讲支撑这件事的组织机制——把可复用的工作流写成 skill，用 Git 统一分发，用遥测和 eval 持续度量质量与成本，再让智能体自己提交缺陷报告形成改进闭环。对研发效能团队最有参考价值的三点：按调用次数和会话成本核算 skill 的投入产出；用「带 skill / 不带 skill」的盲测消融证明 skill 是否真的有用；把 eval 通过率 70%–90% 视为比 100% 更可信的信号。

**出处**　Uber Engineering，Andrew Mager、Jason Varbedain、Uday Kiran Medisetty，2026 年 11 月 23 日。本文为全文级中文译述，覆盖原文全部章节、数据与图表，非逐句对照翻译；封面与图 1–16 引用 Uber 原图，图内保留英文；文中数据均来自 Uber 内部，迁移到其他组织需重新测量。

[Uber 英文原文](https://www.uber.com/us/en/blog/optimizing-ubers-agent-skills/)

![封面｜Agent Skill 的生命周期：构建、运行、度量、评估、改进，并回传指标](https://tb-static.uber.com/prod/udam-assets/b29beefb-8911-453c-9ba8-48a35e635241.png)

## 企业级规模下的 Skills

Skill 是一个 Markdown 文件，教智能体把一件事做好，例如创建 PR、查询指标库、起草周度销售演示。它是 Uber「软件工厂」的构件之一。Uber 的共享 skill 市场在一年内从 2025 年 10 月的 1 个增长到 5,000 多个；数千人写过 skill，智能体在一个月内调用 skill 超过 100 万次。

文章面向的是「在全公司推智能体」的团队，讨论在规模化之后如何管理 skill 的质量、效率与分发。

### 什么是 Skill

文章引用 Stratechery 访谈中的说法：**企业的最小单元是工作流**。一个 skill 封装一条工作流——一份 runbook、一个团队流程，或某位专家手工完成的任务；短到改一个 Linear issue，长到完整的根因分析。

落到磁盘上，skill 是一个目录：`SKILL.md` 包含名称、描述和指令正文，目录里还可以放脚本和参考文件。智能体分三阶段加载，所以几千个 skill 平时只占很短的描述：

- **始终加载**：名称和描述，约 100 tokens，智能体据此判断是否适用。
- **运行时加载**：正文，控制在 500 行以内。
- **按需加载**：脚本和引用文件，只由需要它的步骤读取。

格式细节见开放的 Agent Skills 规范。

![图 1｜skill 目录的组成，以及智能体在何时加载每一部分](https://tb-static.uber.com/prod/udam-assets/89c986e4-39b8-4614-9264-8286ff3c016c.png)

### 分发 Skill

Uber 建了一个统一存放 skill 的地方，让人们在任何使用智能体的地方都能方便加载。最终选的是 Git 仓库：版本控制天然给每个 skill 带来负责人、评审记录和历史。

![图 2｜13 周内每周结束时市场中的 skill 数量（按仓库 Git 历史统计），从 3 月的约 200 个增至 11 月的约 5,300 个](https://tb-static.uber.com/prod/udam-assets/3f82fd86-c131-4b55-968a-c2d636c3e791.gif)

![图 3｜2026 年 3–10 月每月的 skill 调用记录，覆盖 Claude Code、OpenCode、Codex 和 Uber 内部智能体](https://tb-static.uber.com/prod/udam-assets/36f50039-bd1d-41d5-aa87-36b5f7b867e7.gif)

### 三层结构：Always-On、Core、Domain

目标是在让团队容易写 skill 的同时，默认就给出一套经过筛选的 skill。为此市场分成三层，skill 越往上「毕业」，对质量和效率的要求越高；毕业要通过 eval 系统和一组质量、有用性检查。

- **Domain（领域层）**：任何团队都可以开自己的区域，自行指定评审人。一次同事批准加一次 CI 检查即可发布，不必等中央团队。
- **Core（核心层）**：被广泛使用的 skill 上移到这里，由专门团队推荐、测试和维护，平台团队评审每一次变更。
- **Always-on（常开层）**：约 100 个通用 skill 默认对所有人开启，新员工第一次运行就有 skill 可用，覆盖 PR、代码评审、工具接入、数据分析和聊天。

![图 4｜三层仓库结构、一个 plugin 包含什么，以及从 PR 到发布的路径](https://tb-static.uber.com/prod/udam-assets/4932563b-842e-428b-9e0d-f1d77aa37814.png)

### 一次安装，适配所有智能体

同时支持多种智能体是 Uber 的策略：人们可以按任务挑智能体，但每个智能体定义和安装 skill 的方式不同，这本身就是挑战。

Uber 做了命令行工具 **aifx**，最初用于启动各类智能体和保存配置，后来扩展到 skill 管理、MCP 调用、可观测性和更复杂的配置。它抹平各平台在 skill 定义与安装上的差异：`aifx plugin add data-analyst` 安装一组 skill（plugin），`aifx skill add analyze` 安装单个 skill。选择记录在一个配置文件里，对下一个启动的智能体生效；工程师在 Claude Code、Codex、OpenCode 之间切换，拿到的 skill 是一样的。

![图 5｜aifx 的作用：一个命令行工具，让每个智能体获得相同的登录、skill 和工具](https://tb-static.uber.com/prod/udam-assets/786b980d-751a-48eb-ad1b-eadb6fce8e0c.png)

aifx 按层合并配置：公司默认 → 团队或项目 → 仓库 → 工作目录 → 用户 → 本地覆盖。后面的层只追加 plugin，不删除，因此一个仓库可以声明所有在此工作的人都需要哪些 plugin。aifx 还管理 persona，为工程师、数据科学家等角色设定默认 skill 与配置。

![图 6｜aifx 合并六层配置，并把结果写入各智能体自己的配置](https://tb-static.uber.com/prod/udam-assets/dbdbfd97-5b89-4065-96a8-bf2f1fe1a3dd.png)

![图 7｜一个 skill 仓库同时供给笔记本上的智能体、托管智能体和搜索；遥测、eval 与 papercut 回流给 skill 负责人](https://tb-static.uber.com/prod/udam-assets/b5255b08-a424-4553-800b-071101c80c65.png)

### Skill 发现

有几千个 skill 之后，帮用户找到当前工作流需要的那一个变得关键。Uber 有一个 `find-skills` skill，能在具体工作流中找到并下载一个高质量 skill 并直接运行。为支持发现，skill 会按 eval 结果和运行次数分类排序，并生成 skill 质量分。借助内部上下文图，任何员工都可以问：“给我看看同事在用、但我还没用的 skill。”

![图 8｜find-skills 如何为一条 prompt 找到合适的 skill](https://tb-static.uber.com/prod/udam-assets/2d672d66-dc60-4a21-8076-66bb18750f1e.png)

## 度量 Skill 的使用

Uber 通过 hooks 里的 OpenTelemetry 采集使用数据。hook 发送会话开始、工具调用等事件，从而记录 skill 使用及其上下文。当工具是 skill 时，事件记录 skill、版本，以及是智能体自行选择还是用户输入了命令。事件**不包含 prompt 文本，也不包含文件内容**。

![图 9｜使用事件从智能体 hook 到展示页面的路径](https://tb-static.uber.com/prod/udam-assets/c5aee2e0-80a5-4969-acbd-7efcfcde17d6.png)

基于这些事件，Uber 为每个 skill 计算 12 项指标，包括使用量、留存、触达、共同使用、成本、上下文、延迟、错误、工具、版本、智能体、安装量。作者可以在 Skill Workshop 的 skill 页面看到这些数据。

![图 10｜每个 skill 度量的 12 项指标及其用途](https://tb-static.uber.com/prod/udam-assets/1cfdcad7-5217-45d1-98c1-518392d783be.png)

## 永无止境的 Skill 质量之战

Skill 构造的是智能体工作流，带来的挑战与智能体本身不同：它要在许多上下文、许多智能体环境中工作，而且所在领域常常需要人的判断才能定义正确答案。Uber 因此做了 **Skill Workshop**（一个 plugin 加一个 Web 应用），用系统化评估和用户反馈来改进 skill。

### Skill Workshop

Skill 有生命周期：作者写、测试、通过市场发布，再根据真实运行结果改进，直到更好的模型让它不再必要而退役。Skill Workshop 为每个阶段提供工作流，并为每个 skill 提供展示用量、成本和质量的一页。

Skill 从一份定义清楚的 spec 开始。模型和 harness 变化很快，Uber 希望迭代实现时 skill 仍能工作。spec 保留 skill 的意图：目标、输出、必需或禁止的工具动作，以及面向的智能体和模型。Skill Workshop 从 spec 编译出 `SKILL.md` 和测试 prompt；模型或智能体变化时重新编译。

![图 11｜Skill Workshop 中某个 skill 的页面，以及三组工作流](https://tb-static.uber.com/prod/udam-assets/6b3dd082-4f93-4638-94ac-b079654061bd.png)

### 把 Skill 代码异味固化成 Linter

Uber 把「什么让 skill 变好」「什么导致问题」、研究论文和模型提示指南提炼成一组通过/不通过的检查，例如：

- 步骤编号，每步只有一个目标，并至少有一种软验证方式。
- 每一步给出具体命令，并在进入下一步前检查结果。
- 有错误恢复章节，用表格列出失败与修复方式；“再试一次”不算。

### Evals

只靠用户手工反馈判断 skill 质量不够，Uber 建了多套评估系统。评估集的经验配比是：约 **5% golden cases**（固定输入、带工具 mock 的期望输出）、**80% 真实 prompt 按轨迹评分**、**15% 由作者判断**。

**轨迹（trajectory）** 指的是智能体有没有按正确顺序使用正确的 skill 和工具，而不是只看最终结果。

![图 12｜一个 eval 套件里的 prompt 构成，以及测试 skill 变更的三种环境](https://tb-static.uber.com/prod/udam-assets/afeb8adb-1e5f-4a49-a43e-489266dbd11a.png)

eval 在三个地方运行：后台智能体平台 **Minion**、托管沙箱 **AES**、远程开发机 **devpods**。干净环境暴露了对作者本机的依赖——和「在我机器上是好的」一样，远程运行发现许多 skill 默认了特定文件路径、缓存文档或某个 OS 工具。

面对非确定性的 LLM，eval 仍是半科学半艺术。Uber 的经验是：**通过率 70%–90% 的 eval 往往比 100% 通过更有用**，因为答案本就会变，裁判也会变；100% 通过可能意味着 eval 没有捕捉到答案中有意义的差异。

判断一次运行是否通过，本质是判断什么重要：措辞要完全一致吗？哪些变化不改变含义？智能体自己未必能稳定区分，可能纠结无关细节，或在格式正确时漏掉重要幻觉。LLM-as-judge 最好配有人工标注集，因此 Uber 把 eval 套件拆成 golden 集、真实 prompt 和作者构造的用例。轨迹 eval 看智能体尝试做什么、为什么，以及是否按预期调用 MCP 或流程；结果分和轨迹分合起来评估 skill 与工作流的变更。

### 少即是多：消融测试

Uber 在隔离的评估环境里，用相同输入测试删掉一些文字甚至整个 skill 的影响。对完整 skill，用真实 prompt 分别在带 skill 和不带 skill 时重放，由盲评裁判选更好的答案；也会测试只用 skill 的一部分、删掉部分文字。随着模型变强，多余的文字反而可能降低表现。

![图 13｜带 skill 的答案在盲评中获胜的比例（含 95% 区间）；阴影带是该 skill 的噪声带，即两次相同运行之间的差异](https://tb-static.uber.com/prod/udam-assets/a3534e1f-9372-4325-b3a6-f618e6202fb3.gif)

多数情况下，skill 让会话更成功、成本更低。收益最大的，是那些给智能体提供了它本来难以找到或理解的东西的 skill，例如内部文档的访问方式、执行数仓查询的方法。

## Skill 的效率

有了 skill，任务成本变得可度量，Uber 因此把更多会话迁到使用 skill。不用 skill 的会话可能反复试探、找错表、把工具用错；用了 skill，得到正确答案的概率和效率都会提高。在一个数仓查询 skill 的消融测试中：**不带 skill 的会话多 13% 的工具调用、工具错误多 6 倍、耗时多 31%，带 skill 的答案在 87% 的对比中胜出。**

但改进答案的 skill 本身也有成本：指令、工具调用及结果都进入上下文，会话后续每一轮都要为这份上下文再付费。Uber 度量每个 skill 的成本，并用几种杠杆压降。

### 如何度量一个 Skill 的成本

智能体会累计会话至今的花费，遥测定期上报这个总额。调用某个 skill 时，取调用前最后一次上报的总额和调用后第一次上报的总额相减，差额就是这次调用的成本；按天汇总同一 skill 的所有调用，再除以调用次数，得到**每次调用的成本**。这个数字包含模型读取 skill、随后的轮次，以及该时间窗内的其他工具调用。

### 降本杠杆

- **用代码代替 LLM。** 永远不变的步骤交给脚本，模型不必读每个工具定义和每条结果，即下文的 Code Mode。
- **在更便宜模型的 worker 里运行 skill。** skill 派发一个子智能体，调用方选模型：查资料用小模型，关联结果用最强模型。大的工具结果留在 worker 上下文，只返回有界摘要，后续轮次无需再付费。前后对比中，一个 tracing skill 的每次调用成本**下降 46%**；一个部署 skill 第一周没有下降，所以每次变更后都要单独度量。
- **在 skill 加载时运行脚本。** 当结果很小且总是需要时，加载阶段就跑脚本并把输出放进 prompt，省掉一轮模型调用。一个搜索 skill 从 4 轮降到 3 轮，平均成本**下降 32%**。
- **把轮询移出主会话。** 在花费最高的 20 个会话中，通知类轮次约占支出的三分之一。9 月，Uber 把 PR 监控 skill 的轮询移到了更便宜模型上的后台智能体。

### Code Mode：能用代码就别用 LLM

对确定性工作，代码比 LLM 更便宜、更好测、结果更好。Uber 发现大量 token 花在重复的工具调用上：执行查询再读响应、收发 Slack 消息、搜索远程代码、收发邮件。智能体只要结果，却必须让所有内容经过 LLM 和上下文窗口，又慢又贵。

默认的 MCP 安装让情况更糟：每次调用都会载入所有工具的完整描述。工具超过 100 个时，用户还没输入任何内容，这些定义就占了 **50,000–70,000 tokens**。

受 Cloudflare Code Mode 启发，Uber 把这些流程改写成通过网关调用工具的脚本。这样不必预载工具定义，智能体只读取结果中需要的部分，最常用流程的成功率、延迟和成本都得到改善。code-mode skill 变成包，其他 skill 按名称引用，从而复用高质量实现；同时 Uber 在 skill 和 plugin 中禁用 `.mcp.json`，因为那会安装工具服务器。

![图 14｜安装工具服务器会把每个工具定义和每条结果都放进上下文窗口；code mode 下 skill 通过 aifx 和工具网关调用工具，结果写入文件。token 数为估算](https://tb-static.uber.com/prod/udam-assets/ef76a4b3-96c4-4bef-bc3b-829d7be65f0a.png)

## 持续改进：让智能体自己提 Bug

模型升级会改变智能体读指令的方式，工具会改参数，harness 会改变 skill 的运行方式——skill 失效的原因有一千种。

智能体自己知道什么时候不顺利：帮助文本列出了某个参数却被工具拒绝、命令在出错时退出码为 0、本该有匹配的查询返回空。它可以自己找绕行办法，但那会浪费大量 token 和宝贵的上下文。Uber 于是加了一个常开的 skill，让智能体把感受到的摩擦和 bug 记录下来，形成一个改进飞轮。

### 问题发现

每份报告叫一个 **papercut**，由常开的 `log-papercut` skill 提交，因此每个智能体都看得到、无需被要求就能上报。这不仅改善未来的会话，也改善出问题的那次会话：智能体知道自己可能正陷在问题里，并可以查别人是怎么绕过去的。

报告写明智能体尝试了什么、期望什么、实际发生了什么、什么能修好它。所有报告进入一个按 skill 分组的看板。

![图 15｜智能体在 10 月提交的一份 papercut（有删节）。结构化字段用于归并同类报告，并告知负责人要修什么](https://tb-static.uber.com/prod/udam-assets/b44b7f6b-bb13-4c05-95e0-47c63948f8e7.png)

### 问题解决

papercut 的流程是：归并 → 分诊 → 根因分析 → 修复 → 评估 → 批准；修复上线后还要度量并检查回归，之后才关单。

![图 16｜papercut 闭环，两个标注步骤需要人参与](https://tb-static.uber.com/prod/udam-assets/37d243db-8241-41ae-addc-964e4777d7cf.gif)

三个月内智能体提交了数千份 papercut，数百份已修复，修复既改善了工作流，也显著节省了 token。文中举了几个例子：

- PR skill 把 commit message 写进共享临时文件，两个并发会话互相改写了对方的 PR。
- 部署 skill 让智能体传 epoch 秒，而 API 期望毫秒。
- 配置查询 skill 在请求的 key 不存在时，静默返回了另一个属性。
- PR skill 在一个没有 CI 的仓库上等待检查，反复 404 直到超时。

### 人的信号

除了智能体自己的判断，Uber 还有人工评分。在自动软件维护工厂 **Fawkes** 中，代码维护类 skill 会开 PR；如果 PR 收到评论、被反复迭代或未被接受，这些信号就用来改进生成它的 skill，被卡住的 PR 会进入 eval 集。这个模式在多个领域和智能体平台上复用。

## 结论

Skill 把文字变成了动作和结果。随着使用量增长和模型进步，重点从「写 skill」转向「skill 的质量与评估」。仍未解决的问题是：如何评估那些很少运行、但很重要、又难以判断好坏的工作流。Uber 正在试验轨迹与 trace 分析、结构化人工标注和 skill 路由。

## TL;DR

- **所有智能体共用一种格式、一个仓库。** 同一个 skill 能跑在笔记本上的 Claude Code、OpenAI Codex、OpenCode、Cursor，以及 Uber 所有托管智能体里；aifx 统一负责身份、skill 分发和 MCP 连接。
- **Skill 不只是给工程师的。** 工程师用它做根因分析、实验解读；其他团队用它取指标、起草待评审文档、帮助客户经理回答商家支持问题。
- **先写 spec。** `spec.md` 记录 skill 必须产出什么、支持哪些智能体和模型，Skill Workshop 据此编译 `SKILL.md` 和测试 prompt，使 skill 在模型与智能体变化时仍能工作。
- **少有时胜过多。** 用真实 prompt 做带/不带 skill 的重放，盲评选优；模型变强后，一些 skill 需要修改甚至退役。
- **从 trace 改进 skill。** 作者在真实会话中查看失败点，把每次失败转成测试用例。
- **智能体自己提 Bug。** 遇到坏参数、错误文档或静默错误时，智能体提交结构化的 papercut，由负责人处理。

## 下一步

- **Skill A/B 测试**：在大规模上评估 skill 内容，对重要 skill 做金丝雀发布。
- **按模型编译 skill**：不同模型家族的提示指南不同，`SKILL.md` 的措辞与长度也应不同。
- **采集和处理 skill 运行的 trace**：按 eval 给会话建索引，记录用户想做什么、发生了什么、是否遇到问题、是否成功，并总结可改进点与值得推广的做法。
- **Skill 推荐**：告诉用户应该用哪些 skill。
- **任务路由器**：用户只在一个入口提出需求，由系统路由到正确的智能体、skill 和上下文。

## 致谢

本文工作横跨 AIFX、Developer Platform、Michelangelo、uReview、Fawkes、Minion、Cortana、Agentic XP 和 Conan AI 团队，以及 Uber 数百位编写和维护市场 skill 的工程师。封面图由 ChatGPT 生成。文中引用 Shyam Sankar 与 Ben Thompson 的 Stratechery 访谈（2026 年 10 月 29 日）中「企业的最小单元」一说。
