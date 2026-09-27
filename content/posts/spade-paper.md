+++
title = 'Paper Reading: Spade Synthesizing Assertions for Large Language Model Pipelines'
date = 2024-03-01T20:26:39-05:00
draft = false
categories = ['Learned Data Systems']
tags = ['Paper Reading']
+++

## SPADE: Synthesizing Assertions for Large Language Model Pipelines

Synthesizing Assertions

Pipelines

> 合成断言、流水线

## ABSTRACT

将大型语言模型（LLM）用于定制、重复数据 pipeline 的操作具有挑战性，特别是由于其**不可预测和潜在的灾难性故障**。认识到这些错误的不可避免性，我们将重点放在识别 LLM 在作为数据生成 pipeline 的一部分重复使用时可能**生成错误响应**的情况。我们提出了 spade，一种自动合成断言的方法，可以识别不良的 LLM 输出。SPADE 分析提示版本历史，**创建候选断言函数**，然后**选择满足覆盖率和准确性要求的最小集合**。在对九种不同的实际 LLM pipeline 进行测试时，与简单的基线相比，spade 有效地**减少了 14% 的断言数量**，并将**错误失败率降低了 21%。**

## INTRODUCTION

在数据 pipeline 中使用大型语言模型（LLM）引起了广泛关注[17]。这种热情很大程度上归功于其简单性：无需大型标注数据集，只需用自然语言提示 LLM，就能轻松创建一个**自动化的智能 pipeline **，在数秒内执行任意数据生成任务。然而，将 LLM 用于无监督、重复或大规模的数据生成任务却面临着巨大的挑战[34]，例如**潜在的错误、不恰当的反应或幻觉**[44, 57]。

> Data Management For Large Language Models: A Survey，没太理解引用这篇论文的意思，是在预训练、微调时管理数据
>
> 用 LLM 制作应用时，始终无法避免 potential errors, inappropriate responses, or hallucination

考虑使用 LLM 生成个性化电影推荐说明的电影流媒体平台。开发人员可能会编写类似的提示模板： "根据用户的以下信息，为用户应该观看 {movie_name} 的原因编写个性化说明： {personal_info}"这样的提示模板，针对多个用户-电影配对执行。模板化变量代表的信息将在运行时注入 pipeline ，使 pipeline 能够为各种输入生成数据。从理论上讲，这种提示似乎足够了，但开发人员在对一些示例输入进行测试时可能会发现一些问题：LLM 可能会引用用户从未看过的电影，引用敏感属性（如种族或民族），甚至出现响应过短等基本问题。

为了解决这些缺陷，有人提出了在 LLM pipeline 中加入断言 assertions 的框架，**以便在到达最终用户之前过滤掉不良生成**[37, 49]。鉴于 LLM 错误的不可避免性[23]，此类断言是成功部署生成数据的 LLM pipeline 的关键。然而，开发人员发现为自定义 LLM pipeline 编写断言非常困难 [35]。面临的挑战包括：**预测所有可能的 LLM 故障模式**；使用各种**规范方法**（如 Python 函数或 LLM 调用）编写断言耗时；断言（尤其是涉及 LLM 调用的断言）**必须精确**；以及许多 LLM pipeline 开发人员缺乏软件工程专业知识或编码经验 [26，57]。此外，如果断言过多或不具信息性，开发人员在监控其结果时可能会不知所措

在本文中，我们发现了一个新问题，即如何为任何利用 LLM 的数据生成 pipeline **自动生成一组好的断言**。对于 LLM 响应，一个断言返回真（即成功）或假（即失败），而一组断言则返回单个断言的联结。一组 "好 "的断言具有最小的重叠，允许开发人员在错误的成功和错误的失败之间进行权衡。我们将问题分解为两个部分--候选断言生成和过滤 candidate assertion generation and filtering --并提出了 spade（ System for Prompt Analysis and Delta-Based Evaluation 图 1）。

![图 1](https://s2.loli.net/2024/03/03/lgh8FAURaXPZzoE.png)

> delta 在这里是 diff 的意思？
>
> candidate assertion 和 filtering 都是怎么实现的？看上去还是 LLM 实现的

对于候选断言的生成，我们可以考虑直接询问 LLM "为 x 提示编写断言"，但这可能无法满足开发人员的要求。我们建议从 prompt deltas（即两个连续提示符版本之间的差异）中生成**候选断言**，这通常表示 LLM 的特定故障模式。例如，开发人员可能会添加类似 "避免花哨语言 "的指令，从而促使断言检查响应语言。我们对来自 LangChain（一家帮助人们构建 LLM pipeline 的初创公司）用户的 19 个自定义数据生成 pipeline 和 pipeline 提示版本历史进行了分析，从而形成了一个 a taxonomy of prompt deltas。Spade 首先在分类法中自动对提示脱节进行分类，然后将 Python 函数（可能包括 LLM 调用）合成为可用断言。我们公开发布了 SPADE 的这一组件，展示了这些断言的潜力，它在金融、医药和 IT 等 10 多个领域有 1300 多个用途 [46] 。

> 在 [langchain](https://blog.langchain.dev/spade-automatically-digging-up-evals-based-on-prompt-refinements/) 上的 SPADE 例子

我们对候选断言的分析发现了冗余、不准确和大量函数，仅几个 prompt deltas 就往往超过 50 个。冗余源于对提示相似部分的重复完善，或提示指令含糊不清（如 "返回一个简洁的回复"）。减少冗余并不简单，即使对工程师来说也是如此，因为断言可能涉及精确度不同的 LLM 调用，这就需要一个自动过滤组件。一种方法是使用**开发人员标记**的 LLM 响应来估算每个断言的**错误失败率**（false failure rate FFR），并剔除每个超过开发人员定义的 FFR 阈值的断言。然而，其余断言的 FFR 可能会累计超过这个阈值，冗余可能会持续存在。我们的研究表明，选择一小部分断言来满足故障覆盖率和 FFR 标准是 NP-hard 的。也就是说，我们可以用**整数线性规划**（ILP）来表达这个问题，并使用 ILP 求解器来找出解决方案。然而，我们发现标记的响应可能无法代表所有故障模式，从而导致遗漏有价值的断言。例如，在我们的电影推荐场景中，如果开发人员标注样本中的所有回复都低于 200 字，那么能够正确验证 LLM 生成的注释是否少于 200 字的断言就会被放弃。为了扩大覆盖范围，可以使用**主动学习**和**弱监督**方法为每个候选断言采样和标注新的 LLM 提示-响应对，但这可能很昂贵，或者非程序员无法使用。我们引入了断言子块（assertion subsump-tion）作为确保全面覆盖的一种方法：如果一个断言不包含另一个断言的故障模式，那么两个断言都会被选中。因此，SPADE 会选择一组最小的断言，并**遵守故障覆盖率、准确性和子集约束**。总的来说，我们做出了以下贡献：

> Today’s research release of ChatGPT is the latest step in OpenAI’s iterative deployment of increasingly safe and useful AI systems. Many lessons from deployment of earlier models like GPT-3 and Codex have informed the safety mitigations in place for this release, including substantial reductions in harmful and untruthful outputs achieved by the use of reinforcement learning from human feedback (RLHF).
>
> https://openai.com/blog/chatgpt?ref=blog.cg-wire.com
>
> OpenAI 用 RL 和人工监督，来规避一些安全问题和减少不真实输出

**Prompt Deltas** for Candidate Assertion Generation：我们发现 **prompt version history 是数据生成 LLM pipeline 正确性标准的丰富来源**。我们根据 19 个具有版本历史的 LLM pipeline 的推断出了断言标准分类法，并将其公布于众。我们公开发布了一款工具，用于为任何 pipeline 生成候选断言，通过 1300 多次部署为自动生成断言的实用性提供了早期证据，同时也为筛选这些断言提供了机会。

**Filtering Method** that Requires Little Data: 我们证明，在覆盖故障模式和满足准确性要求的前提下，选择**最小的断言集**是一个 NP-Hard。我们将这一问题表述为 ILP。鉴于断言可能是冗余和不准确的，而且 LLM 开发人员可能不具备工程专业知识或足够的数据来使用现有的数据驱动方法进行 ML pipeline 测试（如训练模型），我们引入了断言归并作为在低数据量环境下覆盖率的**新型替代方法**。

Empirical Study: 我们介绍了 spade，这是我们为数据生成 LLM pipeline 自动生成断言的系统。对于九个具有提示版本历史的 LLM pipeline ，我们收集了带标签的 prompt-response pairs（其中八个已开源）。

![alt text](image.png)

> https://spade-beta.streamlit.app/

在所有 pipelines 中，Spade 生成的断言都具有 good failure coverage 和 few false failures（即无法做出良好响应）。我们基于子假设的 ILP 优于不考虑断言间交互的简单基线，减少了 14% 的断言，降低了 21% 的错误失败率。

## IDENTIFYING CANDIDATE ASSERTIONS

我们的第一个目标是生成一组 candidate assertions。我们将介绍 prompt deltas 如何为候选断言提供信息，并解释如何从中得出候选断言。

### Prompt Deltas

单步 LLM 流水线由一个提示模板 P 组成，该模板通过对输入元组 𝑡 进行格式化，生成一个提示 𝑝，并返回一个响应 𝑟。P 可以**有很多版本**，这取决于开发者如何**迭代他们的 prompt template**。让 $P_0$ 表示空字符串，即第 0 个版本，让 $P_i$ 表示模板的第 i 个版本。在第 1 节的电影推荐示例中，假设有 7 个版本，其中 P7 如下：

> Write a personalized note for
> why a user should watch {movie_name} given the following informa-
> tion about the user: {personal_info}. Ensure the recommendation
> note is concise, not exceeding 100 words. Mention the movie’s genre
> and any shared cast members between the {movie_name} and other
> movies the user has watched. Mention any awards or critical acclaim
> received by {movie_name}. Do not mention anything related to the
> user’s race, ethnicity, or any other sensitive attributes

我们将 prompt delta $ΔP_{𝑖+1}$ 定义为 $P_i$ 与 $P_{i+1}$ 之间的差值。具体来说，提示 ΔP 是一组句子，其中每个句子都标记为添加（即 "+"）或删除（即"-"）。例如，表 1 显示了多个版本的 ΔPs。ΔP𝑖 中的每个句子都由添加（即 P𝑖 中的新句子，P𝑖-1 中不存在）和删除（即 P𝑖-1 中的句子，P𝑖 中不存在）组成。对句子的修改由删除和添加表示--例如，表 1 中的 ΔP6 包含了从 P5 中添加到句子中的一些新指令。如表 1 最右边一栏所示，ΔP𝑖 中的每一个添加都表示可能的断言标准。

![](https://s2.loli.net/2024/03/03/PCyvBcO8q2YRGLn.png)

### Prompt Delta Analysis

我们分析了从 LangChain 用户那里收集到的 19 个 LLM pipeline，每个 pipeline 由 3 到 11 个历史提示模板版本组成。这些 pipeline 涵盖各种任务，从生成工作总结到自定义问答聊天机器人。附录 B 中的表 6 显示了这些 pipeline 的摘要，包括每个 pipeline 的描述和提示版本的数量。对于每个 pipeline ，我们将提示符（即 ΔP𝑖）分为不同类型--例如，指示 LLM 在每个回复中包含一个新短语（即包含），或指示 LLM 以某种语气回复（即定性标准）。我们对分类进行了 4 次反复修改，最终形成了图 2 中的分类法。提示版本的分类注释数据集可在网上找到 3。大多数提示符都属于数据整合（即添加占位变量）和工作流程描述（即添加指示让 LLM 更准确地执行任务）。在表 2 中，我们使用引言中的电影推荐 pipeline 示例，展示了分类法中每个类别的提示分隔符示例

![](https://s2.loli.net/2024/03/03/gXvSiduVAmU8Y6x.png)

当从自动识别的提示分隔线中得出断言候选项时，断言质量显然取决于**识别类别的准确性**。因此，我们确认了 GPT-4 对提示分界点的正确分类（截至 2023 年 10 月）。我们为 19 个 pipeline 的所有提示版本分配了基本真实类别，GPT-4 的 F1 得分为 0.8135。用于从提示分段中提取类别的提示详见附录 C

> 什么是 F1 score？ 在二进制分类和信息检索系统的统计分析中，F-SCORE 或 F-MEASIE 是对预测性能的量度。

### From Taxonomy to Assertions

利用我们的分类法，SPADE 使用 LLM 为 prompt deltas 分类并生成断言，详见附录 C。对于每个 ΔP𝑖 ，SPADE 都会提示 LLM 为断言提出尽可能多的标准，每个标准都与分类标准的类别一致 (e.g., Figure 3) 标准 Criterion 被宽泛地定义为对示例进行操作并评估为 "真 "或 "假 "的一些自然语言表达式（例如，"check for conciseness"）。Spade 会**分析每个 ΔP𝑖** 而不是只分析最后一个提示版本，原因有以下几点：开发人员通常会删除提示中的指令以降低成本，同时期望获得相同的行为[35]；提示包含固有的歧义，并暗示了评估某些标准的多种方法；如果只分析一个版本，复杂的提示可能会**导致遗漏断言**。因此，**分析每个 ΔP𝑖 会增加生成相关断言的可能性**

![](https://s2.loli.net/2024/03/03/VImk3LJovBM4uAf.png)

对于每个 delta，spade 都会收集已识别的标准，并再次提示 LLM 创建 Python 断言函数。合成函数可以使用外部 Python 库，也可以针对复杂的标准向 LLM 提出二进制查询。在函数合成时，LLM 会被指示，如果标准指定得很模糊或可以解释，例如 "check for conciseness"，它就可以生成多个函数，每个函数都对标准进行评估。在这个简洁性示例中，LLM 可以返回多个函数--一个将回复分成句子并确保不超过（例如）3 个句子的函数，一个将回复分成单词并确保不超过（例如）25 个单词的函数，或者一个将回复发送给 LLM 并询问回复是否简洁的函数。这一步骤的总体结果是候选函数的多集合 𝐹 = {𝑓1, ..., 𝑓𝑚} 。

我们的方法采用了两步流程，因为事实证明，**将任务分解为多个步骤可以提高 LLM 的准确性** [21, 53, 55]。不过，值得注意的是，随着 LLM 日新月异的发展，**一步式流程可能已经可行**。此外，虽然我们的**分类法**现在可以指导 LLM 生成断言，但未来的 LLM 可能会通过**人类反馈的强化学习来隐式地学习这些类别**[12]。不过，了解与候选断言相关的基于分类学的类别可能有助于过滤它们

> Designing LLM Chains by Adapting Techniques from Crowdsourcing Workflows. arXiv preprint arXiv:2312.11681 (2023)
>
> Chain-of-thought prompting elicits reasoning in large language models. Advances in Neural Information Processing Systems
>
> 任务分解为什么可以提高 LLM 的准确性？而且现在 ChatGPT 就是用的 Human Feedback RL 来增强的 Deep reinforcement learning from human preferences

### LangChain Deployment

> 比较了早期版本和后期的区别，吹了一下自己生成的断言有多么好，涵盖了多少个领域，达到了 "perfect"
>
> 提到了一个总结文章的例子，具有 14 个 prompt 版本，对他 SPADE 后，可以有一些断言，必须要有一些关键词等等
>
> 但这些断言并不是正则匹配，而是字符串匹配，我觉得不太合理。
>
> 文中也提到说许多断言可能是不正确的，怎么知道断言是正确的呢？本文的实验可能也不够，于是采用了自动化 filtering

## FILTERING CANDIDATE ASSERTIONS

在这里，我们解决了第 2.4 节中确定的**冗余和不正确断言**的问题，特别是在具有大量 prompt 版本的 pipeline 中。 过滤此候选集不仅可以提高部署断言以在生产中运行时的效率，而且还可以减少开发人员的认知开销

### Definitions

将 𝑒𝑖 视为 LLM pipeline 在某些输入上的端到端执行（即运行）的示例。 令 𝐸 为所有此类示例运行的集合（该集合未预先提供，我们将很快处理）。 我们定义一个断言函数 𝑓 : 𝐸 → {0,1}，其中 1 表示成功，0 表示失败。 令 𝐹′ ={𝑓1, 𝑓2, . 。 ., 𝑓𝑘} 是一组 𝑘 断言。 当且仅当一个例子 𝑒𝑖 **满足 𝐹' 中的所有断言**时，该集合才认为它是成功的。 具体来说

> 一些数学定义，定义了 Coverage 和 False Failure Rate

### Coverage Problem Formulation

> 目标是找到最小的集合 minimal set of assertions 满足所有例子？
>
> 找到一个 Coverage(𝐹′) ≥𝛼, FFR(𝐹′) ≤𝜏
>
> 找到限制，就变成了一个优化问题，集合覆盖问题

我们将此 ILP 的解决方案称为 $spade_{cov}$。 简单地说，对于 𝜏 =0 和 𝛼 =1，这个问题是 NP-hard 的，通过简单地从集合覆盖归约，并且是 NP 问题，因为它可以用 ILP 形式表示

### Subsumption Problem Formulation

到目前为止，我们**假设开发人员愿意提供一组全面的标记示例运行** 𝐸′，在开发人员不愿意这样做的设置中，并且 𝐸′不包括 𝐸 中的所有故障类型，spade_cov 可能会 忽略 𝐹 中的有用断言，这些断言只能捕获 𝐸 \𝐸′ 中的失败——如第 4 节中的经验所示。我们最初考虑使用主动学习 [6] 为每个断言采样更多的 LLM 响应，并使用弱监督来标记响应 [36 ]。 然而，这种方法对于最先进的 LLM 来说成本高昂，并且**需要大量的手动工作来平衡每个断言的失败和成功示例**，确保有意义的 FFR 并避免由于代表性不足的故障类型而排除断言。 对于这种设置，我们还引入了包含。 假设所有候选断言函数涵盖尽可能多的故障模式，我们的目标是选择 𝐹′ ⊆ 𝐹，使得 𝐹 \𝐹′ 中的断言包含在 𝐹′ 中。 形式上，如果 𝑆 中函数的合取逻辑上意味着 𝑆 和 𝑓 中函数的合取，则一组函数 𝑆 包含某个函数 𝑓 。

> 太多定义了，可能主要是在解决这个集合覆盖吧

## EVALUATION

我们首先讨论 LLM pipelines 和数据集（即 𝐸′）； 然后，我们讨论方法和指标并展示我们的结果。 实验代码、数据集和 LLM 响应托管在 GitHub 上

### Pipeline and Dataset Descriptions

我们对 9 个 LLM pipeline 进行了评估，其中 8 个来自 LangChain Hub（LLM pipeline 的开源集合），以及 1 个专有 pipeline 。 六个 LangChain Hub pipeline 帮助开发了 prompt delta taxonomy（第 2.2 节），但在创建分类法后，从 spade 的 Streamlit 部署（第 2.4 节）添加了两条 pipeline 。 专有 pipeline 是 fashion pipeline，它为活动提供服装建议。 之所以包含此 pipeline ，是因为它使用了 LLM 培训中未包含的数据及其实际部署，展示了 Spade 的 real-world 性。 虽然我们使用真实的用户提示模板和历史记录（3 到 16 个提示版本之间），但我们构建了自己的一组示例提示-响应对和标签 (𝐸′) 进行测试。 LangChain Hub pipeline 的两个数据集来自 Kaggle，而其他数据集是使用 Chat GPT Pro（基于 GPT-4）综合生成的。 例如，对于使用 LLM 审查拉取请求的 codereviews pipeline ，我们要求 Chat GPT 生成涵盖各种编程语言、应用程序类型和差异大小的占位符值。 我们对 8 个 LangChain pipeline 的 LLM 回复进行了标记，以评估他们是否符合提示说明。 响应分为 GPT-3.5-Turbo 和 GPT-4。 对于 fashion pipeline ，标签是由开发人员在相应的启动时完成的。 表 3 提供了每个 LLM 流程和数据集的详细信息。 我们已经开源了 LangChain Hub 8 个 pipeline 的所有数据

### Method Comparison and Metrics

和之前一样，令 𝐸′ 为示例提示-响应对的数据集，以及响应是好（即 1）还是坏（即 0）的相应标签。 令 𝜏 为 FFR 阈值，𝐹 为 SPADE 第一步产生的候选断言集（第 2 节）。 如果候选函数 𝑓 导致某些示例 𝑒 出现运行时错误，我们表示 𝑓 (𝑒) =0（即失败）。 我们所有的代码都是用 Python 编写的，使用 PuLP Python 包来寻找 ILP 的解决方案。 我们使用默认的 PuLP 配置，它使用 CBC 求解器 [18]。 我们评估了 spade 的三个版本： • $spade_base$ 选择 𝐹 中的所有函数 𝑓，其中 FFR ({𝑓 }) ≤𝜏 • spade_cov 是第 3.2 节中定义的 ILP 的解 • spade_sub 是第 3.3.1 节中定义的 ILP 的解 让 𝐹′ 代表任何版本的 spade 所选择的断言集合。 我们测量四个指标：

(1) 所选断言的分数（即 |𝐹′|/|𝐹|）

(2) 排除的非包含函数的分数（即 |𝐺|/|𝐹|，其中 𝐺 ={𝑔 |𝑔 ε𝐹 \𝐹′ 和 𝐹′ ̸=⇒ 𝑔})

(3) 错误失败率（定义 3.2）

(4) 𝐸′ 的覆盖率（定义 3.1）

此外，spade_sub 成功的一个重要方面是包含的有效性 所有断言对之间的评估。 由于我们**没有包含的基本事实**，因此我们关注**精度**，计算为正确识别的包含对在 LLM 识别的所有包含对中的比例。 **我们不评估召回率**（GPT-4 是否识别了每一个可能的包含），因为为每个管道标记可能数万个断言对是不切实际的。 此外，**精确度**比召回率或准确性更重要，因为即使识别一些包含，spade_sub 也能用比 $spade_base$ 更少的选定断言来实现解决方案。

> 这里评估精确率，但却忽略召回率，感觉是论文不够全面的地方。
>
> 精确率：TP/ (TP + FP)，我觉得有故障中的真故障 / （我觉得有故障且真故障 + 我觉得有故障但没故障）
>
> 召回率：TP / (TP + FN), 我觉得有故障中的真故障 / （我觉得有故障且真故障 + 我觉得没故障但真故障）
>
> 能否理解为，精确率区分真假正确，召回率针对样本中有多少正确的是你给出来的。所以一般两者都要一起考虑？

### Results and Discussion

使用 GPT-4 评估所有 pipeline 的归并结果，**平均精度为 0.82**，如表 5 所示，证实了其有效性。 为简单起见，我们将所有管道的覆盖率和 FFR 阈值设置为相同（𝛼 =0.6，𝜏 =0.25）。 我们在表 4 中报告了三种方法的结果。例如，考虑 codereviews 管道，它使用 LLM 来审查任何代码存储库的拉取请求。 这里， $spade_base$ 选择 20 个断言， $spade_base$ 选择 2 个断言， spade_sub 选择 15 个断言。 通过选择更多函数，spade_sub 可确保包含所有未包含的函数。 所有三种方法都遵守 𝐸′ 覆盖约束，但 $spade_base$ 在 9 个管道中有 4 个违反了 FFR 约束。 平均而言，与 $spade_base$ 相比，spade_sub 选择的断言数量减少了约 14%，并且 FFR 显着降低，相对于 $spade_base$ 减少了约 21%。 spade_cov 平均排除了 44% 未包含在 𝐹′ 中的函数。 我们随后讨论不同 spade 实现之间的权衡

> 结果和例子都显示了比较高的 precision，而且用的是 GPT-4
>
> 其中 code review pipeline 的一些 assertion 会回去调用 LLM，询问是否包含代码修改的建议，review 是否精简，是否聚焦于该 PR
>
> 剩下的没仔细看，很多结果并不是很令人信服，尤其是对三个不同的 SPADE (base 选择所有的 f，cov 选择最小的集合，sub 是 Subsumption Constraints 限制的)

不同版本 SPADE 的结果，𝛼 =0.6 和 𝜏 =0.25。 勾号和 x 标记表示是否满足 𝛼 和 𝜏 约束。 每个条目都是该管道的候选断言总数的一部分（括号中是绝对数量）。 spade_cov 总体上选择最少的断言。 spade_sub 在优化包含时选择最少的断言

𝜶 and 𝝉 Threshold Sensitivity：Spade 中 ILP 求解器的解决方案的可行性取决于所选的 𝛼 和 𝜏 阈值。 如果找不到可行的解决方案，开发人员可能需要以二分搜索的方式调整这些值。 在我们的例子中，所有 9 个 LLM 管道都产生了 𝛼 = 0.6 和 𝜏 = 0.25 的可行解决方案。 然而，𝐸′ 的小尺寸使得 spade_cov 对 𝛼 特别敏感。 在管道中，我们观察到 1 到 5 个断言覆盖了 60% 的失败。 例如，spade_cov 只为电子邮件管道选择了一个断言

FFR Tradeoffs: 考虑到对于三个 LLM 管道，为 spade_base 和 spade_sub 选择的函数比例之间的差异小于 10%，人们可能想知道 spade_sub 的复杂性是否值得。 spade_sub 通常更可取，因为 spade_base 无法始终满足 FFR 阈值 𝜏。 我们观察到，随着提示版本的增加，断言的数量也会增加，从而对 spade_base 产生不利影响。 一组的最坏情况 FFR 是各个 FFR 的总和，如公式 (2) 所示。 因此，在存在大量独立断言的情况下，总 FFR 很可能会超过阈值。这个问题在 fashion 和 lecturesummaries 管道中很明显，尽管 67 和 32 个断言中的每一个都单独满足 FFR 约束，但 spade_base 的总 FFR 分别达到 0.88 和 0.53。 实际上，如果将 spade 部署在交互式系统中，其中 spade 可以实时观察每个 LLM 调用（例如，作为 OpenAI API 的包装器），则大量的提示版本进一步需要基于整体 FFR 过滤断言 。 这强调了对更复杂的 spade_cov 或 spade_sub 方法的需要

### Limitations and Future Work

Improving Quality of LLMs: 虽然 LLM（封闭式和开源）正在迅速改进，并且我们没有明确研究 spade 的 prompt engineering strategies，但补充的研究想法是**探索此类策略或微调小型开源模型以生成断言**。 此外，我们提出将 subsumption 作为覆盖范围代理，但没有探索 prompt 工程甚至非 LLM 策略（例如，断言来源）来评估包含。 尽管 LLM 取得了进步，但 SPADE 的过滤阶段对于减少冗余和确保准确性仍然至关重要，特别是因为断言可能涉及 LLM。

Collecting Labeled Examples: 获取标记数据（𝐸′）很困难。 我们的大多数数据集的提示版本很少（只有提交给 LangChain Hub 的版本），但实际上，开发人员可能会对其提示进行数十或数百次迭代。 未来的工作可能涉及通过 LLM API 包装器进行被动示例收集或收集开发人员对断言的反馈。 对不同类型的断言进行优先级排序并在 spade 中将这些优先级形式化是另一个需要探索的领域。 此外，在有限的 𝐸′ 下评估 FFR 估计的准确性，并探索在缺乏大型标记数据集的情况下提高 FFR 准确性的方法（例如，通过预测驱动的推理 [2]），为未来的工作提出了一个有趣的领域

Supporting More Complex LLM Pipelines: 我们的研究重点是单提示 LLM pipeline，但更复杂的管道（例如涉及多个提示、外部资源和人工交互的管道）为每个管道组件自动生成断言提供了机会。 例如，在检索增强生成管道中[28]，甚至可以在生成 LLM 响应之前将断言应用于检索到的上下文。

## RELATED WORK

Prompt Engineering: 对于非技术用户 [57] 和技术用户 [34, 47] 来说，Prompt 工程很困难，原因有几个：提示措辞 [4, 29] 或指令或上下文的顺序 [30] 的微小变化可能会显着影响输出。 此外，随着 LLM 在 API 下发生变化（即提示漂移），输出可能会在开发人员不知情的情况下发生变化 [9]。 帮助提示管理和实验的工具和论文不断出现，甚至使用 LLM 来编写提示 [3,11,54,55,59]。 此外，部署的提示引入了新的挑战，例如“用更少的标记平衡更多的上下文”和“争论提示输出”以满足用户定义的标准[35]。 我们的工作并不明确地专注于帮助开发人员创建更好的提示，但它可以通过推荐的断言间接支持开发人员改进提示。

ML and LLM Evaluation: 众所周知，评估和监控已部署的机器学习模型具有挑战性 [33, 45]。 在部署环境中评估 LLM 更具挑战性，因为 LLM 通常用于生成任务，其输出是自由形式的[13]。 一些 LLM 管道类型，例如使用检索增强生成管道的问答 [28]，可以使用标准化的自动化指标 [14, 39]，但其他类型则因未知指标和缺乏标记数据集而面临挑战 [8, 35, 56]。 通常，组织依靠人类评估者来评估 LLM 的输出[19,35,52]，但最近的研究表明 LLM 可以通过详细的“记分卡”进行有效的自我评估[7,26,58]。 然而，编写这些记分卡可能具有挑战性 [35]，从而激励自动生成的评估者

LLMs for Software Testing: LLM 越来越多地用于软件测试，主要用于生成单元测试和测试用例[27,40,48,50,51]。 研究探讨了 LLM 的激励策略、幻觉和不确定性如何影响代码或测试的准确性 [10,15,16,32]。 我们的工作是互补的，并利用 LLM 为 LLM 管道生成基于代码的断言

Testing in ML Pipelines: **ML pipelines 在生产中很难管理**。 许多有关机器学习测试的文献都是通过分析数据质量 [5, 22, 42, 43] 或来源 [31, 41] 来验证结构化数据。 ML 测试平台通常提供自动实验跟踪和防止过度拟合 [1, 38]，以及数据分布调试工具 [20]。 特定于模型的断言通常需要人类规范[25]，或者至少需要大量数据来训练学习断言[24]。 **LLM 链或管道是一类新型 ML 管道**，LLM 本身可以用很少的数据生成断言。 最近的一项研究强调了测试“copilot”类产品的 LLM 管道的难度：开发人员希望确保准确性，同时避免过度使用资源，例如运行数百个断言 [35]——这激发了我们的断言过滤方法。

## CONCLUSION

我们的工作引入了自动生成断言以捕获 **LLM 管道中的故障的新问题**，以及包含两个组件的解决方案：首先，它 synthesizes candidate assertions； 然后，它会 filter 它们。 我们提出了断言合成的快速编辑分类法，通过与 LangChain 的集成和部署展示了其潜力。 我们将用于高精度覆盖故障的最佳断言集的选择表示为整数线性程序（ILP）。 我们提出了断言包含来涵盖数据稀缺场景中的故障，并将其纳入我们的 ILP 中。 我们的自动生成断言系统 Spade 在 9 个真实数据生成 LLM 管道上进行了评估。 我们已公开我们的代码和数据集以供进一步研究和分析

> 没有理解 SPADE base, cov, sub 之间的区别，只是选择的集合不同吗

> The paper presents SPADE, which is a method to detect incorrect outputs from large language models (LLMs) by automatically synthesizing assertions when developing LLM pipelines (such as codereviews and lecturesummaries). SPADE analyses the prompt version histories and creating candidate assertion functions based on the prompt deltas. Then, SPADE selects a minimal set of assertions or a subset that satisfies the coverage and accuracy requirements, meaning that they can catch most of the errors without causing too many false failures. The paper also uses a pruning technique to remove redundant or contradictory assertions from the final set. Finally, The paper evaluates SPADE on 9 real-world LLM pipelines and shows that it can reduce the number of assertions by 14% and decrease false failures by 21% compared to simpler baselines.

> 1. The paper proposes a novel formulation of synthesizing assertions for LLM pipelines, which takes into account the prompt version histories, the coverage and accuracy requirements, and the trade-off between assertion complexity and number. It also provides an open-source implementation of SPADE and deploy it on the Langchain.
> 2. The SPADE can effectively reduce the number of assertions and false failures compared to simpler baselines, and that spade-generated assertions can help developers improve their LLM pipelines. Spade reduces the number of assertions by 14% and the number of false failures by 21% compared to simpler baselines.
> 3. It evaluates SPADE on nine real-world LLM pipelines, and demonstrates its effectiveness in reducing the number of assertions, increasing the coverage and accuracy of assertions, and identifying bad LLM outputs.

> 1. Like all the applications built on LLM, it relies on the quality of the language model, the ability of LLM correct classificationand the availability of prompt version histories. If the prompt is too large or the language model performs bad, the assertions (python code or string match) might not work.
> 2. The paper does not compare SPADE with Reinforcement Learning from Human Feedback(ChatGPT is enhanced with RLHF to filter inaccurate text and harmful output). SPADE also ignored the very important metrics, recall rate, in the experiment.
> 3. SPADE does not discuss the different effects brought by different assertion implementation methods. For example, most assertions are string matching and many of them might even not work, which may cause some assertions to fail and increase FFR.

> I would try to first compare SPADE with other existing methods for LLM quality control, such as RLHF. Then explore more appropriate assertion implementation methods instead of simple string matching, and discuss the impact of different assertions on FFR. And during the experiment, try to deeply explore the impact of different coverage rates on recall and precision. Finally, try to generalize SPADE to more language models, such as smaller models with less parameters, making the LLM pipeline construction cheaper
