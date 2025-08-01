---
title: Agentic RAG数据合成流水线 2
icon: solar:palette-round-linear
createTime: 2025/07/14 16:37:14  
permalink: /zh/guide/agenticrag_pipeline2/  
---

# Agentic RAG 数据合成流程

## 1. 概述

**Agentic RAG 数据合成流程**是一个端到端的框架，用于：  
- 支持基于强化学习的 Agentic RAG 训练。
- 从提供的文本内容中生成高质量的问题和答案对。

该流程只需要文本上下文即可生成高质量的问题和答案，用于后续训练。

---

## 2. 数据流与流程逻辑

### 1. **输入数据**

该流程的输入数据包括以下字段：

* **text**：各种文本内容

这些输入数据可以存储在指定的文件中（如 `json` 或 `jsonl`），并通过 `FileStorage` 对象进行管理和读取。在提供的示例中，默认数据路径被加载。在实际使用中，你可以修改路径以加载自定义数据和缓存路径：

```python
self.storage = FileStorage(
    first_entry_file_name="../dataflow/example/AgenticRAGPipeline/pipeline_small_chunk.json",
    cache_path="./cache_local",
    file_name_prefix="dataflow_cache_step",
    cache_type="jsonl",
)
```

### 2. **原子问答生成**

#### 2.1 **问答生成**

流程的第一步是使用 **原子任务生成器** 算子（`AtomicTaskGenerator`）从大型数据集中分别生成问题、参考答案、精简的参考答案、可替代（验证）以及在提供原始文档下LLM对问题的回答。

**功能：**

* 从一组文本上下文中生成问题、参考答案、精简的参考答案、可替代（验证）以及在提供原始文档下LLM对问题的回答。

**输入**：原始文本内容

**输出**：问题、参考答案、精简的参考答案、可替代（验证）以及在提供原始文档下LLM对问题的回答

```python
llm_serving = APILLMServing_request(
                    api_url="https://api.openai.com/v1/chat/completions",
                    model_name="gpt-4o-mini",
                    max_workers=500
        )

atomic_task_generator = AtomicTaskGenerator(
                            llm_serving=llm_serving
                        )
result = atomic_task_generator.run(
            storage = self.storage.step(),
            input_key = "text",
        )
```

---

### 3. **问答生成质量评估**

#### 3.1 **F1打分器**

流程的第二步是使用 **F1打分器** 算子（`F1Scorer`）为精简的参考答案与提供原始文档下LLM对问题的回答之间的 F1 分数进行评估。这一步确保每个已构造的问题，在正确地文档检索下，所回答的答案能被合适地给予reward，保障强化学习的训练质量。

**功能：**

* 为精简的参考答案与提供原始文档下LLM对问题的回答之间的 F1 分数进行评估。

**输入**：参考答案、 提供原始文档下LLM对问题的回答
**输出**：F1 分数

```python
f1_scorer = F1Scorer(
            prediction_key="refined_answer",
            ground_truth_key="golden_doc_answer"
        )
result = f1_scorer.run(
            storage=self.storage.step(),
            output_key="F1Score"
        )
```

---

## 3. 运行流程

运行完整流程：

```python
import pandas as pd
from dataflow.operators.eval import *

from dataflow.operators.generate import (
    AtomicTaskGenerator,
    DepthQAGenerator,
    WidthQAGenerator
)

from dataflow.operators.filter import *
from dataflow.utils.storage import FileStorage
from dataflow.serving import APILLMServing_request, LocalModelLLMServing
from dataflow.core import LLMServingABC

class AgenticRAGEvalPipeline():

    def __init__(self, llm_serving=None):

        self.storage = FileStorage(
            first_entry_file_name="../dataflow/example/AgenticRAGPipeline/pipeline_small_chunk.json",
            cache_path="./agenticRAG_eval_cache",
            file_name_prefix="agentic_rag_eval",
            cache_type="jsonl",
        )

        llm_serving = APILLMServing_request(
            api_url="https://api.openai.com/v1/chat/completions",
            model_name="gpt-4o-mini",
            max_workers=500
        )

        self.task_step1 = AtomicTaskGenerator(
            llm_serving=llm_serving
        )

        self.task_step2 = F1Scorer(
            prediction_key="refined_answer",
            ground_truth_key="golden_doc_answer"
        )
        
    def forward(self):

        self.task_step1.run(
            storage = self.storage.step(),
            input_key = "contents",
        )

        self.task_step2.run(
            storage=self.storage.step(),
            output_key="F1Score"
        )

if __name__ == "__main__":
    model = AgenticRAGEvalPipeline()
    model.forward()
```

---