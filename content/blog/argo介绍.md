---
title: RAG 分享
date: 2025-02-04T08:15:33.113Z
---

# 一、背景
检索增强生成 Retrieval Augmented Generation（RAG）指在生成响应之前，在LLM输入中引用外部文档从而增强输出。RAG在改善LLM回答的可追溯性解决幻觉问题、克服LLM窗口大小限制、为LLM补充未训练的领域/最新知识等方面有着重要的作用。RAG从最开始简单的Retreive-Read流程到现在每个模块迭代都趋于复杂，都是为了LLM能生成更高质量的内容。下面将结合以往的工作和近期调研对一些重点模块进行介绍。

# 二、Embedding Model 向量模型
对于领域RAG应用，对向量模型进行SFT会十分显著地提升召回率。训练目标是将相关的query-document pair映射到向量空间较近的位置，反之不相关的映射到较远的位置，即最小化负样本pair的相关分以及最大化正样本pair的相关分。

**样本**

负样本的选取策略对模型效果有着比较大的影响。可以采用in-batch negatives 和 hard negatives相结合的方式。

in-batch negatives表示同一训练batch内的其他样本，即可以作为该样本的负样本。

hard negatives可解释为“难”的负样本，即检索器容易混淆，误判为相关的负样本。这种类型的负样本挖掘方式可以采用其他base模型的prediction结果，即将其他base模型误判的样本作为hard negatives。这里可以采用bm25负样本、base embedding模型负样本，或是混合搜索（第三章介绍）负样本。

只使用领域数据集进行SFT可能会导致其在通用检索能力上有明显下降（训偏），因此训练时可能需要混合一些通用数据。

需要注意训练数据集应该尽可能使用Query-Document而非Query-Answer类型，RAG的应用场景为Query-Document，混入Query-Answer数据集会导致效果损失。

bi-encoders为了降低复杂度、节省内存，通常会复用同一个encoder，而在query和document前加上例如"query:"、"document:"的prefix能够有效区分，一些论文中表明这样的做法能够略微提升召回精度。

**训练**

训练框架可以使用deepspeed(simlm)、megatron-lm等，都有现成的embedding模型训练代码。

训练损失函数采用listwise loss。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526149-56c77e31-e57a-4f4b-892f-afeb244a7a44.png)

训练时batch size越大越好，通常采用显存允许的情况下最大的mini-batch。这里可以使用共享embedding的技巧，在一个训练步的不同mini-batch间利用all-gather共享embedding，这样做能够提升batch-size至原来的data-parallel倍。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526112-9cfb9137-0c45-4e4e-9f96-54559225d2b6.png)

**推理**

目前绝大多数embedding模型都采用transformer encoder（BERT）结构，推理时可以直接使用faster transformer加速。

**评测**

好的评测能够指导召回器的迭代，目前公开通用中文检索数据集评测集可以采用[C-Mteb](https://huggingface.co/spaces/mteb/leaderboard)，但对不同业务来说，如果能够提供业务数据集进行评测，会对检索器的效果提升有较大的帮助。

# 三、Hybrid Search 混合搜索
混合搜索（Hybrid Search）指的是将多种搜索策略或算法结合在一起。在RAG中，较为流行的做法是将基于关键词的搜索和基于语义和向量的检索相结合，因为两者都各有优劣，结合后能力上可以互补。两者都能够通过向量表示，根据向量中数据的分布情况可以分为稀疏向量和稠密向量，因此混合搜索也可以称为稀疏与稠密相结合的搜索方法。

| 搜索策略 | 优势 | 向量表示 | 典型模型 |
| --- | --- | --- | --- |
| 关键词搜索 | 专有名词、术语的检索能力强 | [0, 0, 0, 0, 0, 0.2, 0, 0, 0, 0.24, 0.3, 0, 0, 0, 0, ...]<br/>每个元素表示该位置上的token的分数，由于大部分位置为0，称之为稀疏向量，也可以用indice,values进行压缩：{'indices': [1358668261, 1991459541, 930853874, 924587953], 'values': [0.52, 0.52, 0.43, 0.43]} | BM25（TF-IDF） |
| 向量检索 | 具备语义检索能力 | [-0.04,-0.06,0.007,-0.06,0.32, ...]<br/>每个token上都有非零值，称之为稠密向量。 | Transformer Encoder（BERT) |


检索策略的混合方式采用分数加权，在每种策略分数分布一致的情况下较为简单的经验公式为：

_最终相关性分数 = 0.8 * 向量检索相关性 + 0.2 * 关键词检索相关性_

如有精力，也可以根据不同数据集特点进行定制化微调。

将上述的加权检索分数嵌入已有的RAG系统中有多种实现方式：

增加每种策略的召回数量，如最终期望召回5个document，则每种策略可以召回20个document，加权打分后得到top5。这种方法可能会漏召回一些两种策略表现都较好的documet。

直接修改向量检索库中的相关分计算函数，可以得到准确的混搜top5。

经过实际的评测，关键词搜索+向量检索的混合检索**在几乎所有数据集效果都好于单一的关键词搜索或向量检索**。

# 四、Reranker 重排
重排可以类比为CTR场景的精排模型，耗时但精度更高。狭义的重排模型仅关注相关性，而现代的重排模型则集成了多样性等更复杂的策略。

**相关性重排**

与embedding模型通常会采用的bi-encoders不同，相关性重排模型一般会采用cross-encoders，encoder的输入为query-document pair，encoder上接一个线性层（分类器）直接输出pair的相关度（0-1）。

重排之所以更耗时主要是因为召回仅需要向量化一个query，而重排则需要对每一个候选pair进行推理，精度更高则是因为attention能够同时关注query和document，即单独的query和document不再有固定的向量表示。

**多样性重排**

先选择最相关的doc，再选择最不相关的，重复这个过程，以使模型喂入的上下文具有多样性，回答的生成更广泛、更具有深度。

**“中间缺失”重排**

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526066-20989d29-32b7-49ff-99b1-1f09670f33db.png)

在头和尾放置最相关的doc，其理论来源基于研究：[Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/pdf/2307.03172.pdf)，即当相关信息出现在输入上下文的开始或结束时，性能通常最高

# 五、RAG Evaluation 系统评测
目前RAG评测的维度通常为检索质量（Retrieval Quality）和生成质量（Generation Quality）两部分，评测的方向则通常为

上下文相关性（Context Relevance）检索到的上下文是否与问题相关

忠实性（Faithfulness）生成的回答是否与上下文保持一致性而没有冲突

回答相关性（Answer Relevance）回答是否直接针对提出的问题，有效地解答核心询问

目前市面上已经有一些自动化的RAG评测系统例如[RAGAS](https://github.com/explodinggradients/ragas)、[ARES](https://github.com/stanford-futuredata/ARES)、[RECALL](https://arxiv.org/pdf/2311.08147.pdf)等。这里选择开源star数最多的RAGAS进行介绍。

**RAGAS**

RAGAS是一个非常强大的自动化RAG评测框架，主要借助LLM来完成评测、测试集生成等一系列功能。RAGAS既可以评测单一的模块 Component-Wise Evaluation（召回、生成），又可以端到端 End-to-End Evaluation 地评测RAG整体的效果。

评测：仅需按照格式提供结果集（包含Question、Ground Truth、Answer、Contexts），并指定需要评测的指标，就可以用几行代码生成评测结果。其实现方法是调用能力较强的LLM（GPT4），利用few-shots给出几个打分例子后利用LLM来进行打分。

```plain
from ragas import evaluate
 
result = evaluate(
    fiqa_eval["baseline"].select(range(3)), # selecting only 3
    metrics=[
        context_precision,
        faithfulness,
        answer_relevancy,
        context_recall,
    ],
)
 
result
```

测试集自动生成：仅需给出参考文档，就能够用几行代码自动生成RAG评测测试集。实现方法同样是利用few-shots，调用LLM实现。

```plain
from ragas.testset import TestsetGenerator
 
testsetgenerator = TestsetGenerator.from_default()
test_size = 10
testset = testsetgenerator.generate(documents, test_size=test_size)
```

测试集的生成使用了创新的方法进行数据增广。首先根据给定的document生成seed question，然后针对实际应用场景中的问题多样性，对seed question进行不同方向上的改写，以下举一些例子：

seed question：种子问题

```plain
{
    "context": "The Eiffel Tower in Paris was originally intended as a temporary structure, built for the 1889 World's Fair. It was almost dismantled in 1909 but was saved because it was repurposed as a giant radio antenna.",
    "output": "Who built the Eiffel Tower?"
}
```

reasoning question：逻辑推理问题，将问题复杂化，重写为一个需要多步推理的问题。

```plain
{
    "question": "What is the capital of France?",
    "context": "France is a country in Western Europe. It has several cities, including Paris, Lyon, and Marseille. Paris is not only known for its cultural landmarks like the Eiffel Tower and the Louvre Museum but also as the administrative center.",
    "output": "Linking the Eiffel Tower and administrative center, which city stands as both?"
}
```

multi_context question：多上下文问题，即回答问题需要不止一个document。

```plain
{
    "question": "How do you calculate the area of a rectangle?",
    "context1": "The area of a shape is calculated based on the shape's dimensions. For rectangles, this involves multiplying the length and width.",
    "context2": "Rectangles have four sides with opposite sides being equal in length. They are a type of quadrilateral.",
    "output": "What multiplication involving equal opposites yields a quadrilateral's area?",
}
```

conditional question：条件性问题，给问题增加限制条件。

```plain
{
    "question": "What is the function of the roots of a plant?",
    "context": "The roots of a plant absorb water and nutrients from the soil, anchor the plant in the ground, and store food.",
    "output": "What dual purpose do plant roots serve concerning soil nutrients and stability?"
}
```

compress question：压缩问题，将问题缩短，变得更不直接但保留相同的意思。

```plain
{
    "question": "What is the distance between the Earth and the Moon?",
    "output": "How far is the Moon from Earth?"
}
```

# 六、RAG 应用平台
目前市面上的RAG应用平台（GPT-Store，Coze，Dify等）面向无代码的LLM应用开发者，能够以非常低的成本构建出一个智能助理，编排出一个LLM应用。

以构建一个面试官应用来介绍智能助理的构建路径：

编写人设和提示词：

```plain
我想让你担任{{jobName}}面试官。我将成为候选人，您将向我询问{{jobName}}开发工程师职位的面试问题。我希望你只作为面试官回答。不要一次写出所有的问题。我希望你只对我进行采访。问我问题，等待我的回答。不要写解释。像面试官一样一个一个问我，等我回答。当我回准备好了后，开始提问。
```

添加知识库：对于专业领域的面试官应用，需要添加专业知识库从而生成高质量的面试问题。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526086-250584eb-f57d-444a-a95c-5dde4b58d5e9.png)

选择模型和模型参数：选择应用的底座LLM模型，设置相应的模型参数。

![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526099-9bea0359-cde1-40de-8dc3-1e6cd0eb08b0.png)

调试和发布：通过端到端或是对局部模块的测试，不断迭代调整各个模块的参数、配置，提升LLM应用的效果。

面试官应用：[https://udify.app/chat/lPY7ZSIGgpbykTSE](https://udify.app/chat/lPY7ZSIGgpbykTSE)

一个LLM应用是否能够有好的效果很大程度上取决于迭代测试过程，开发和测试的时长可以达到2:8甚至1:9，因此下面重点从迭代测试的角度看一个LLM应用平台的各个模块应该如何设计。

| 模块 | 配置 | 测试迭代流程 |
| --- | --- | --- |
| 知识库模块 | 知识库的切分策略（分割符，块大小）<br/>知识检索策略（Embedding模型选择，混合检索策略的分数配比，召回个数，召回阈值） | 切分后的文档可视化，可编辑<br/>模拟单条召回测试<br/>自动化批量测试：用LLM为每个文档生成一些问题，再用这些问题整体测试召回率 |
| 模型模块 | LLM选择<br/>LLM配置 | 需要有不同模型/配置下的生成内容对比调试<br/>![](https://cdn.nlark.com/yuque/0/2024/png/34760166/1706509526511-a97f999e-c365-4382-973e-863611b9f4b5.png) |
| 工具模块 | 自定义工具编写（api，参数） | 工具模拟调用调试 |
| 提示词模块 | 人设和提示词（角色，目标，技能，限制） | 有完整的会话内容展示，所见即所得，包含上下文和召回文档<br/>需要有预览页能够一边审查生成结果，一边改善提示词。 |
| 端到端评测 | 以上所有配置 | 单条会话测试<br/>批量生成RAG测试数据，对RAG系统进行不同维度上的打分 |


# 七、RAG展望
目前的RAG系统每个模块都趋于复杂化，为了提升各个模块的效果，都从最初简单的规则演变到了后来复杂策略甚至是基于LLM的执行，这更像是智能体Agent应用，各个模块的动态智能决策成为了演进的关键。例如把Retriever当成和谷歌搜索一样的工具，那么什么时候应该使用Retriever工具、应该如何使用（用什么样的关键词进行检索）都应该由LLM自己作决策。

Multimodal Retrieval Augmented Generation(MM-RAG) 多模态RAG：具备多模态信息(文字、图片、视频）的召回能力，并将多模态信息作为多模态LLM的上下文。简单的思路：参考多模态LLM的训练（fuyu-8b），训练多模态向量模型时，将相似语义的多模态信息被映射到向量空间相对较近的距离，仅在预处理时额外增加图片、视频的"tokenzier"。

# 参考文档
[https://weaviate.io/blog/cross-encoders-as-reranker](https://weaviate.io/blog/cross-encoders-as-reranker)

[https://www.pinecone.io/learn/series/rag/rerankers/](https://www.pinecone.io/learn/series/rag/rerankers/)

[https://towardsdatascience.com/improving-retrieval-performance-in-rag-pipelines-with-hybrid-search-c75203c2f2f5](https://towardsdatascience.com/improving-retrieval-performance-in-rag-pipelines-with-hybrid-search-c75203c2f2f5)

[https://arxiv.org/pdf/2312.10997.pdf](https://arxiv.org/pdf/2312.10997.pdf)

[https://docs.haystack.deepset.ai/docs/ranker](https://docs.haystack.deepset.ai/docs/ranker)

[https://arxiv.org/pdf/2307.03172.pdf](https://arxiv.org/pdf/2307.03172.pdf)

[https://towardsdatascience.com/enhancing-rag-pipelines-in-haystack-45f14e2bc9f5](https://towardsdatascience.com/enhancing-rag-pipelines-in-haystack-45f14e2bc9f5)

