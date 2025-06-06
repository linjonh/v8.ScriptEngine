---
title: "设计评审指南"
description: "本文档解释了 V8 项目的设计评审指南。"
---
请确保在适用时遵循以下指南。

V8 设计评审正式化有多个驱动因素：

1. 让个人贡献者（IC）清楚决策者是谁，并在因技术分歧导致项目无法推进时明确前进的路径
1. 创建一个直接的设计讨论论坛
1. 确保 V8 技术负责人（TL）知晓所有重大变更，并有机会在技术负责人（TL）层面提供他们的意见
1. 增强全球范围内所有 V8 贡献者的参与度

## 概要

![V8 设计评审指南概览](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

重要事项：

1. 假设他人出于善意
1. 保持友善与文明
1. 保持务实

建议的解决方案基于以下假设/支柱：

1. 建议的工作流程让个人贡献者（IC）负责。他们是这一流程的推动者。
1. 他们的指导 TL 负责帮助他们导航领域并找到合适的 LGTM 提供者。
1. 如果某个功能没有争议，几乎不会产生额外的负担。
1. 如果争议很多，该功能可以“升级”至 V8 工程评审所有者会议，在那里决定下一步行动。

## 角色

### 个人贡献者 (IC)

LGTM: 不适用
此人是功能的创建者和设计文档的撰写者。

### IC 的技术负责人 (TL)

LGTM: 必需
此人是特定项目或组件的 TL。通常，这个人是您的功能将接触的主要组件的拥有者。如果不清楚谁是 TL，请通过 [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com) 询问 V8 工程评审所有者。TL 负责在需要时将更多人加入到需要 LGTM 的人员列表中。

### LGTM 提供者

LGTM: 必需
这是需要提供 LGTM 的人。可能是 IC，或者是 TL (M)。

### 文档的“随机”审阅者 (RRotD)

LGTM: 非必需
这是只是审阅和评论提案的一方。他们的意见应被考虑，尽管他们的 LGTM 并非必需。

### V8 工程评审所有者

LGTM: 非必需
被卡住的提案可以通过 [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com) 升级到 V8 工程评审所有者。以下情况可能会导致此类升级：

- LGTM 提供者未响应
- 对设计无法达成共识

V8 工程评审所有者可以推翻非 LGTM 或 LGTM 的决定。

## 详细工作流程

![V8 设计评审指南概览](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

1. 开始：IC 决定开发一个功能/被分配一个功能
1. IC 将早期设计文档/解释文档/一页文档发送给一些 RRotD
    1. 原型被视为设计文档的一部分
1. IC 将他们认为需要提供 LGTM 的人添加到 LGTM 提供者列表中。TL 一定要在 LGTM 提供者的列表中。
1. IC 结合反馈意见。
1. TL 将更多人添加到 LGTM 提供者列表中。
1. IC 将早期设计文档/解释文档/一页文档发送到 [v8-dev+design@googlegroups.com](mailto:v8-dev+design@googlegroups.com)。
1. IC 收集 LGTMs，TL 协助其完成。
    1. LGTM 提供者审阅文档，添加评论，并在文档开头给出 LGTM 或非 LGTM。如果他们给出了非 LGTM，则有义务列出原因。
    1. 可选：LGTM 提供者可以将自己从 LGTM 提供者列表中移除，并/或推荐其他 LGTM 提供者
    1. IC 和 TL 致力于解决未解决的问题。
    1. 如果收集到所有 LGTMs，发送邮件到 [v8-dev@googlegroups.com](mailto:v8-dev@googlegroups.com)（例如通过回复原始线程）并宣布实施。
1. 可选：如果 IC 和 TL 被卡住并/或希望进行更广泛的讨论，可以将问题升级到 V8 工程评审所有者。
    1. IC 发送邮件到 [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com)
        1. 确保 TL 在邮件中抄送
        1. 邮件中包含设计文档的链接
    1. 每位 V8 工程评审所有者都有义务审阅文档，并可选择将自己添加到 LGTM 提供者列表中。
    1. 决定解锁功能的后续步骤。
    1. 如果之后仍未解决阻碍或发现新的、无法解决的阻碍，则进入上一步骤。
1. 可选：如果在功能已获批准后添加了“非 LGTMs”，它们应被视为正常的未解决问题。
    1. IC 和 TL 致力于解决未解决的问题。
1. 结束：IC 继续其功能开发。

始终记住：

1. 假设他人出于善意
1. 保持友善与文明
1. 保持务实

## 常见问题

### 如何决定某个功能是否值得编写设计文档？

以下是一些设计文档适用的指引：

- 涉及至少两个组件
- 需要与非V8项目进行协调，例如调试器、Blink
- 实现需要超过一周的时间
- 是一项语言功能
- 涉及平台相关的代码修改
- 用户可见的更改
- 有特殊的安全考虑或安全影响不明显

如果不确定，请询问TL。

### 如何决定将谁添加到LGTM提供者列表中？

以下是一些指引，说明何时应将人员添加到LGTM提供者列表中：

- 预计会修改的源码文件/目录的OWNER
- 预计会接触的主要组件的专家
- 你的更改的下游消费者，例如当你更改一个API时

### 谁是“我的”TL？

可能是主要组件的OWNER，该组件是你的功能将要涉及的。如果不清楚谁是TL，请通过v8-eng-review-owners@googlegroups.com询问V8工程评审OWNER。

### 哪里可以找到设计文档的模板？

[这里](https://docs.google.com/document/d/1CWNKvxOYXGMHepW31hPwaFz9mOqffaXnuGqhMqcyFYo/template/preview)。

### 如果发生重大变化怎么办？

请确保你仍然获得了LGTM，例如通过提醒LGTM提供者并设定清晰、合理的反对截止时间。

### LGTM提供者没有对我的文档发表评论，我该怎么办？

在这种情况下，你可以按照以下升级路径处理：

- 通过邮件、Hangouts或文档中的评论/分配直接提醒他们，明确要求他们添加LGTM或非LGTM。
- 让你的TL参与进来并寻求帮助。
- 升级至v8[eng-review-owners@googlegroups.com](mailto:eng-review-owners@googlegroups.com)。

### 某人将我添加为文档的LGTM提供者，我该做什么？

V8旨在让决策更透明，并使升级流程更直接。如果你认为设计足够好且应当执行，在表格中你的名字旁的单元格中添加“LGTM”。

如果你有阻塞问题或意见，请在表格中你的名字旁的单元格中添加“非LGTM，因为\<原因>”。你可能需要准备进行新一轮的评审。

### 这与Blink意向过程如何配合工作？

V8设计评审指南补充了[V8的Blink意向+更正流程](/docs/feature-launch-process)。如果你正在推出一个新的WebAssembly或JavaScript语言功能，请遵循V8的Blink意向+更正流程以及V8设计评审指南。通常在发送实现意向之前收集所有LGTM是有意义的。
