+++
title = 'Paper Reading: CatSQL: Towards Real World Natural Language to SQL Applications [VLDB 23]'
date = 2024-04-14T14:19:15-04:00
draft = false
tags = ['Paper Reading']
+++

## CatSQL: Towards Real World Natural Language to SQL Applications

NL2SQL, text to SQL 是很有趣的方向。有 DL 方法也有现在的 LLM 微调。

## ABSTRACT

rule-based 或者 Deep Learning 方法，要么无法通用，或者存在语法/语义错误/无法执行。

本文提出的新框架，注重 Accuracy 和 runtime，提出了 novel CatSQL sketch, which constructs a template with slots that initially serve as placeholders, and tightly integrates with a **deep learning model** to fill in these slots with meaningful contents based on the database schema

对比 sequence-to-sequence 或 sketch-based 方法，不需要生成关键词，有更高准确和更快

对比 sketch-based 方法，更加通用，而且可以用已经填好的词来提高性能。

提出了 Semantics Correction technique, which is the first that leverages database domain knowledge in a deep learning based NL2SQL solution.

Semantics Correction 是 post-processing 后处理，通过 rules 识别和修改语义错误，大大提高了 NL2SQL 的准确度。

实验：single-domain and cross-domain，准确性和吞吐量都提高很多，尤其是在 Spider 基准上，比之前最好的 stoa NL2SQL 要高四分，63x 的吞吐量

## INTRODUCTION

早期 NL2SQL 是 rule-based, 先将 nl parse 成中间态(parsing tree) 然后开发 rules 映射成 SQL 抽象语法树，然后转成 SQL query。这样难以跨域，而且需要大量人力开发规则。

最近借助 Deep Learning 实现 cross-domain adaption，而且提供了大量数据集，比如 Wiki-SQL 和 Spider。部分 DL 方法可以达到 80-90% 的精度，但是对于复杂查询性能会显著下降，不到 50%。以前的方法将其认为是翻译问题，机器翻译问题(seq2seq)，没有利用数据库知识，很容易导致语义错误。

本文是第一个将 DL 结合数据库的，开发了新的 NL2SQL 问题，可以提高性能。

CatSQL, is a deep learning-based approach, which can mitigate the issues of rule-based solutions to generalize across application domains.

1. most existing deep learning approaches are based on a sequence-to-sequence or sequence-to-tree model . These methods do not guarantee the generated SQL queries are **executable** or even syntactically legal. Instead, our approach is a **sketch-based solution**, which relies on our novel CatSQL sketch to generate the SQL query. CatSQL sketch is a template with keywords and slots. These slots initially serve as placeholders. We use a deep learning model to fill in the empty slots to get a final SQL query, which is almost always a legal SQL query

> 完形填空？总是合法的 SQL 语句，那语义能否保证正确呢，结果总是正确的吗。如果结果正确，性能怎么样呢？NL2SQL 会不会知道索引，或者说会不会结合 query optimization 呢？

以前也有 fill in slots 的做法， sketch-based idea

Our CatSQL SQL generation algorithm employs a novel CatSQL sketch, which is **general** enough, and can facilitate the idea of parameter sharing to boost the performance

> 强调了更加 general，而且参数之间可以共享

以前的 DL 没有结合数据库信息，CatSQL 做了语义矫正，To the best of our knowledge, we are the first to incorporate **semantic information** into a deep learning-based NL2SQL solution. We evaluate our CatSQL approach on the well-known Spider dataset, and the results demonstrate that CatSQL is 4 points better than the previous state-of-the-art methods

1. A **novel sketch-based model CatSQL** is proposed to achieve the state-of-the-art performance on various NL2SQL benchmarks;
2. **Semantics Correction** of CatSQL is the first work that adopts database domain knowledge in a NL2SQL solution;
3. **Extensive evaluations** demonstrate that CatSQL significantly outperforms all existing solutions on cross-domain benchmarks such as Spider and WikiSQL;
4. On **single-domain** benchmarks, CatSQL solution also significantly outperforms existing solutions;
5. CatSQL prototype achieves outstanding **runtime performance**: its single query runtime latency can be 2×−20× faster than all baselines; while its **throughput** can be 2.5×−63×higher than the previous approaches

> 主要贡献，CatSQL 新方法基于 sketch-based，语义矫正，大量测试，单领域多领域，运行时、吞吐量性能更好

## OVERVIEW

### Motivating Example

The NL2SQL problem is to translate the **natural language question** into the **corresponding SQL query**. We need to figure out three challenging questions from the natural language description: (1) what tables and columns that will be used in the query; (2) what is the correct query structure; and (3) how to fill in query details and the literal in the query.

> 表和列，query 结构，query details 字面量。

Solutions from the DB community requires designing **rules** for the mapping between natural language tokens into SQL elements, and translating a natural language parsing tree into a SQL’s abstract syntax tree (AST). However, these approaches’ performance decay significantly when they are applied to a new domain of database, that requires redesigning the mapping.

> rules 难以应对新的 domain，新 domain 是什么意思，新的表？还是新的数据？

Deep Learning 训练大模型可以解决 cross-domain 问题，并且性能很好，但是没有利用语义信息生成查询，而且难以处理复杂查询，而且运行时性能和模型参数、复杂度有关。训练需要很多计算。

本文结合了这两种方法，and we demonstrate that our approach performs better than all existing solutions.

Our approach employs a **deep learning architecture**, and we **leverage DB domain knowledge** to develop novel **Semantics Correction** techniques to postprocess the query generated by a neural network to fix obvious semantics errors. In the following, we will first provide necessary background on how a deep neural network NL2SQL solution works, and then we will give an overview of our approach.

### Background

1. Embedding: As an atomic operator, neural networks, such as LSTMs [18] and Transformers [41], can model a sequence of tokens into a sequence of 𝑁-dimensional numeric vectors called embedding

> 这里也提到了 BERT, GraPPa 是另一个表语义分析的预训练模型，应该是专用于 SQL 领域的、表结构的。搜了一下才知道，之前有一篇 RatSQL 就是利用自注意力机制来做的，能知道关系，也就是 Relation-Aware Text2SQL

2. Classifier: Using this operator, we can build a classifier to classify the input sequence into one of 𝐶 categories as follows.

3. Sequence-to-sequence translation:

4. Training:

> 基础知识略

### Existing NL2SQL Approaches

(1) rule-based approaches;

(2) sequence-to-sequence approaches; These approaches typically face the problem of picking a **wrong column or a wrong value**. Therefore, most of the efforts in these works are devoted to tackling this issue

(3) sketch-based approaches. We will briefly explain the basic ideas below

> 具体介绍略，但 rule-based 可以处理复杂结构和 query？而且 Sketch-based 方法并不新，以前也有，比如 SyntaxSQLNet 和 Sqlnet，只不过可能没有结合 RL 或者没有做得那么好。CatSQL 可能就是利用 GraPPa 加上 Sketch-Based 的结合

## CATSQL APPROACH

Column Action Templates

![](https://s2.loli.net/2024/04/15/HbjG14VmxkD3lIR.png)

### CatSQL template

The definition of the CatSQL template is designed to **facilitate the idea of parameter sharing**, which is not adapted by previous sketch-based approaches. In particular, since each of the four CAT clauses can be viewed as a sequence of CATs, we can train one sequence-to-sequence model for all four different clauses. At runtime, once the CATs are predicted, we can simply construct the final SQL query by assembling different CAT clauses. We will explain how our neural network fills a CatSQL sketch in the next section

> parameter sharing 这个概念不太理解，如果说填空之间有联系，能不能用微调后的模型，做一个上下文预测？

### CatSQL query generation

the overall architecture of CatSQL is composed of four components, namely

1. GraPPa embedding network,
2. CAT decoder network,
3. conjunction network, and
4. FROM decoder network. We now explain each of these components

> 就是用 GraPPa 实现了 parameter sharing，看来本质还是上下文预测。想知道和 CatSQL 的区别在哪呢？

![](https://s2.loli.net/2024/04/15/S1TC3I6lcOZ4KGP.png)

### Semantics Correction

**Semantics Correction** technique significantly improves the accuracy by leveraging database domain knowledge. While a deep learning model is effective in understanding the **intention of a question**, sometimes the gene**rated SQL query expresses the same intention**, but is semantically invalid considering the database schema.

> 如果和 CatSQL + GraPPa 区别不大，是不是语义矫正提高了更多的准确性
>
> 如果说 ChatGPT 生成的 SQL 质量较好，是因为 RLHF 吗？

We classify our Semantics Correction rules into three categories: (1) token-level violation, (2) FROM clause revision, and (3) join-path revision.

> 有三个类型可以修正，是 best-effort 而不是保证可以的，也不会把对的写成错的

1. token-level violation 看上去是是类型检查
2. FROM clause revision 有时候 query 会出现一些没出现过的 column 比如 FROM 里没出现过的，此时会直接加到 FROM 里？用最小生成树来判断这些 JOIN 情况。
3. join-path revision 有时候 join 会失败

## SYSTEM IMPLEMENTATION

基于 MySQL v5.7.30

1. Offline processing. 离线预处理，模型训练。甚至用了 redis 存储 literals to embedding (word2vec 而不是 BERT 的原因是更快)
2. Online serving. 在线推理，由于构建了后端，好像可以并行化（吞吐量的由来？）
3. Model details:

## EXPERIMENTS

accuracy, running speed, throughput

### Benchmarks and Baselines

Dataset:

1. cross-domain: WikiSQL, Spider 具有大量的 queries, tables 和 databases
2. single-domain: GeoQuery, Scholar, IMDB...

**Spider** is considered as the hardest NL2SQL dataset currently. Spider supports much richer SQL syntax, and it classifies the dataset into four categories based on their hardness levels

Evaluation metrics:

1. accuracy (i.e., Accex ). 语义上正确的 SQL？还是结果一致？
2. logical form accuracy (i.e., Acclf ). 完全一致的 SQL？
3. executable rate: 本文提出的

Baseline approaches: 文章选了 RAT-SQL，但是是用 GraPPa 预训练的，而 NatSQL 和 SmBoP 选的是 GraPPa 与训练的

### Cross-Domain Evaluation Results

分别测试了没有 parameter sharing 的，没有 semantic correction 的，完整的，

在 Spider 上表现非常好，几乎各项都是第一，没有 parameter sharing 的数据都挺好但看上去 SC 的提升在 easy 提升很明显，但 Extra Hard 就提升没那么多。

而且可执行的概率也特别高，除非没有语义矫正，都是 100%

论文提到 not designed for logical form accuracy？

运行速度也很快，吞吐量也很大。

### Single-Domain Evaluation Results

比 rule based 也要好，但是在 IMDB 上表现比较差，但也比其他都好

> 使用的是 Spider 数据训练，在 IMDB 上表现差是因为很多 join rules 不太适用？

### Throughput analysis

吞吐量高的原因，用了小的模型 GraPPa，所以也更快。而且在 decoding 时，一个 CAT 生成一组 slots，可以大幅并行。可以并行处理 11 个查询，

### Case Study

分析了几个用例，比较有趣的是没法生成 AVG() 这种，而是直接 select average，但是表带有 average 列？

还有就是 Column Action Template 会 flatten conjunction 关系 导致无法生成合适的 SQL 来选取 OR 关系。

论文认为可以多训练 This problem could be possibly handled by either collecting more training data containing queries with complex conjunction relationships, or designing new architecture to predict logical operations explicitly. Both approaches are challenging to implement and require high-quality manual labels. We leave this issue as a further research to study

> 多训练是合理的，不知道微调 GraPPa 能不能预测一些 conjunction 关系

## RELATED WORK

提到了 GraPPa 非常有用

## CONCLUSION

略

> 名字非常好玩，从 RatSQL 到 CatSQL，但实际上论文没有太多比较 RatSQL 甚至 RatSQL 只用了 BERT 来作为 baseline
>
> 也非常创新地利用 RL 从翻译变成了完形填空，而且用 LLM 可以预测上下文的能力进行关联，非常巧妙
>
> 而且用 Semantic Correction 可以后处理这些可能错误的语义，只是还有一些缺陷，文中也说训练也可能可以解决。
>
> 看了下目前 CatSQL 78 分 Acc𝑒𝑥 并不算高的，阿里后面的 DAIL-SQL + GPT-4 和 DAIL-SQL + GPT-4 + Self-Consistency 刷到了 86 分。甚至有匿名的 MiniSeek 刷到了 91 分，可能 GPT 参数更大，还是要更强吧。
>
> 同时最近还有新的 benchmark BIRD-SQL 是阿里和港大一起推出的，
>
> Summarize
>
> The paper propose a novel framework called CatSQL, which tackles the challenge of translating natural language questions into SQL queries. While existing systems like rule-based and deep-learning-based cannot handle cross-domain datasets or donot use the database information, CatSQL creatively combines two approaches. CatSQL leverages the contextual prediction capabilities of language models such as GraPPa to enable parameter sharing based on a sketch-based method and can generate more accurate SQL statements with semantics correction. In addition, CatSQL not only exhibits better accuracy, it is also faster and has greater throughput.
>
> Contributions:
>
> 1. CatSQL innovatively bridges two existing Text-to-SQL approaches (deep-learning and rule-based) and overcomes their problems, for example, rule-based emthods are almost exclusively used for single-domain datasets. CatSQL also turns the existing Deep Learning translation task into a filling task with faster runtime and higher throughput.
> 2. CatSQL applies sketches to construct a SQL statement template with slots and placeholders, and then uses deep learning model like GraPPa to fill the slots with meaningful content based on the database schema. The method is called parameter sharing and can reduce the computational complexity and improve generalization capability and flexibility.
> 3. The paper proposes a semantic correction technique to identify and correct semantic errors in the generated SQL queries with database domain knowledge. The method can detect erros on token level, FROM clause level and join-path level, improving the exact set match accuracy and obtaining a significant improvement over baseline models (RAT-SQL with BERT model).
>
> Limitations:
>
> 1. CatSQL would be very dependent on the performance of deep learning models, although the paper compares Rat-SQL, which is using the BERT model. If the GraPPa model is also used, RAT-SQL as a baseline will also perform better. And from the current Spider Leader Board, the front runners are using larger models like GPT-4, and the score is much higher than CatSQL, at 86 (DAIL-SQL + GPT-4).
> 2. The paper says that CatSQL doesn't focus on accuracy, but no attempt is made to solve the problems encountered in the Case Study such as conjunction and column name confusion by more training, or by switching to a larger model.
> 3. The method Semantic correction is post-processed and could be improved if it can be performed when reasoning or generating SQL statements. In addition, when new benchmarks come out, such as BIRD, which can better evaluate a model's ability to work across data domains, and CatSQL might need to be retrained to achieve better performance.
>
> Improve:
>
> This paper is very innovative and methodologically sound. I might try to replace the language model with a larger one at first, possibly with more parameters capable of stronger context and parameter sharing. And I may do more model training to try to solve the problems in cast study, especially conjunction errors and semantic obfuscation. In addition, perhaps the ChatGPT fine-tuning approach, or Reinforcement Learning Human Feedback (RLHF) could also help CatSQL to achieve higher accuracy. Finally, I would try to run the modified model on more benchmarks for evaluation, especially on the BIRD benchmark.
