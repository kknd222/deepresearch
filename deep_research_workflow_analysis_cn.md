# DeepResearch项目工作流深度解析

---

## 第一部分：项目概述与工作流程

### 1.1 引言

本文档旨在深度解析 **DeepResearch** 项目的内部工作流程。与创建一份通用指南不同，本文将完全基于当前项目的源代码，详细阐述其从接收用户请求到生成深度研究报告的全过程。

DeepResearch 的核心设计理念是：**通过一个由多个专业AI智能体（Agents）协同工作的、可迭代的精细化流程，从海量信息中提炼出高质量、高相关度的核心知识，而非简单罗列搜索结果。**

### 1.2 整体工作流程

通过分析项目代码及架构图 (`img/系统架构图.png`)，我们将项目的核心工作流总结为以下可迭代的闭环：

![系统架构图](https://github.com/cat3399/deepresearch/raw/main/img/%E7%B3%BB%E7%BB%9F%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

1.  **接收请求 (Request Intake)**
    *   用户通过一个兼容OpenAI格式的API接口 (`/v1/chat/completions`) 发起请求。当请求的 `model` 名称中包含 `deep-research` 时，深度研究流程被触发。

2.  **规划 (Planning)**
    *   一个专门的 **“规划智能体” (Planner Agent)** 会分析用户需求，并生成一个初始的、结构化的研究计划（包括要搜索的关键词、语言、搜索目的等）。

3.  **执行 (Execution)**
    *   **搜索 (Search)**: 根据研究计划，系统并发地调用外部搜索引擎API（如 SearXNG）获取初步的URL列表。
    *   **评估 (Evaluate)**: 一个 **“评估智能体” (Evaluator Agent)** 会接收这些URL及其摘要，并根据与用户原始需求的“相关性”对它们进行打分，筛选出最有价值的URL。这是保证信息质量的第一道关卡。
    *   **抓取与压缩 (Scrape & Compress)**:
        *   系统调用爬虫模块，抓取高价值URL的完整网页内容。
        *   一个 **“压缩智能体” (Compressor Agent)** 会接收网页全文，并根据用户的原始需求，将其“压缩”成一段高信息密度的、有针对性的摘要。

4.  **迭代与进化 (Iteration & Evolution)**
    *   系统会判断当前收集到的信息是否足够。
    *   如果信息不足，系统会将**本轮的研究计划和产出的压缩摘要**作为“记忆”，反馈给第一步的“规划智能体”。
    *   “规划智能体”基于这些新信息，生成一个更精确、更深入的**下一轮研究计划**，然后重复第2、3步。
    *   这个循环会持续进行，直到达到预设的最大迭代次数，或者“规划智能体”判断信息已经足够。

5.  **生成报告 (Report Generation)**
    *   循环结束后，一个 **“总结智能体” (Summarizer Agent)** 会整合所有迭代轮次中收集到的、经过压缩的高质量信息片段，生成一份逻辑清晰、内容详实的最终研究报告。

---

## 第二部分：数据抓取与处理的实现

### 2.1 数据抓取（Web Scraping）

#### 实现方式
项目的“数据抓取”环节体现在 `app/utils/url2txt.py` 文件中。它并非一个单一的爬虫，而是一个健壮的、带有故障转移机制的**爬虫调度器**。它会按顺序尝试多种外部爬虫服务（FireCrawl, Crawl4AI, Jina），并优先选择返回内容最丰富的结果。

#### 关键代码
核心逻辑位于 `url_to_markdown` 函数中，它清晰地展示了这种优先级和重试机制。

```python
# app/utils/url2txt.py

# ... (导入和辅助函数)

def url_to_markdown(url: str) -> Optional[str]:
    """
    从给定URL抓取内容并转换为Markdown格式
    """
    attempt_count = 0
    max_attempts = 2
    best_result = ''  # 存储最佳结果
    # ... (处理文件直接下载的逻辑)

    while attempt_count < max_attempts:
        attempt_count += 1
        try:
            # 优先尝试 firecrawl
            if config.FIRECRAWL_API_URL:
                try:
                    result = by_firecrawl(url)
                    if result and len(result) > MIN_RESULT_LEN:
                        return result
                    elif result and len(result) > len(best_result):
                        best_result = result
                except Exception as e:
                    logger.error(f"Firecrawl抓取{url}失败: {str(e)}")

            # 其次尝试 crawl4ai
            if config.CRAWL4AI_API_URL:
                # ... (类似逻辑)

            # 最后尝试 jina
            if config.JINA_API_URL:
                # ... (类似逻辑)

        except Exception as e:
            logger.error(f"抓取过程发生未知错误: {str(e)}")

    if best_result:
        return best_result

    return ''
```

### 2.2 数据工程化：智能内容压缩

#### 处理流程
本项目的数据工程化**并非传统的数据清洗或文本切块（Chunking）**，而是一种更高级的、由AI驱动的**“查询感知型内容压缩” (Query-Aware Content Compression)**。这个流程在 `app/utils/compress_content.py` 中实现。

#### 为何需要这种处理？
1.  **适应上下文窗口**：这是基本目的，确保长篇网页能被LLM处理。
2.  **提高信息密度**：传统切块会引入大量无关信息。而本项目通过LLM进行压缩，只保留与用户查询最相关的核心信息，极大地提高了后续分析的效率和准确性。
3.  **保留上下文关联**：简单的切块会割裂上下文，而LLM在压缩时能理解并保留最重要的逻辑关系。

#### 关键代码
`compress_url_content` 函数是入口，它根据配置选择`by_gemini`或`by_openai`。其核心在于**构建了一个包含用户意图的提示词**。

```python
# app/utils/compress_content.py

def by_openai(url: str, user_input: str, title: str = "", target_type:str = '1'):
    # ... (初始化)

    # 先通过 url_to_markdown 抓取网页全文
    html_content = url_to_markdown(url)

    if len(html_content) > 2000:
        # 核心：构建一个同时包含“用户需求”和“网页内容”的 Prompt
        messages = [
            {'role':'system','content':SYSTEM_PROMPT_SUMMARY},
            {'role':'user','content':f"用户需要的信息:{user_input}\n\n 网页的标题与摘要是{title} 网页的url是{url} 网页内容:{html_content[:70000:]}"}
        ]

        client = OpenAI(api_key=config.COMPRESS_API_KEY, base_url=config.COMPRESS_API_URL)

        # ... (带重试的API调用)

        response_text = completion.choices[0].message.content
        return response_text.strip()
    else:
        # 内容很短，直接返回原文，节省成本
        return html_content
```

---

## 第三部分：大模型的核心交互与分析机制

### 3.1 核心机制：多智能体协同

DeepResearch 的强大之处在于它并非依赖单个LLM，而是构建了一个**多智能体（Multi-Agent）系统**。不同的LLM在精心设计的提示词（Prompt）指导下，扮演着不同的专家角色：

*   **规划师 (Planner)**: 在 `fc_deepresearch.py` 的 `generate_search_plan` 中，负责制定和调整研究计划。
*   **评估师 (Evaluator)**: 在 `search_after_ai.py` 的 `evaluate_relevance` 中，负责筛选有价值的URL。
*   **压缩师 (Compressor)**: 在 `compress_content.py` 中，负责对网页内容进行针对性压缩。
*   **总结师 (Summarizer)**: 在最终环节（由 `process_messages` 触发），负责整合所有信息生成报告。

### 3.2 模型的“记忆”机制：迭代式进化

本项目的“记忆”并非来自模型本身，而是通过**迭代式的输入构建**来实现的。这个机制在 `fc_deepresearch.py` 的主循环 `deepresearch_tool` 中体现得淋漓尽致。

**工作原理：**
1.  **第一次迭代**: `generate_search_plan` 函数被调用，此时 `previous_plan` 和 `previous_results` 为空。它使用 `DEEPRESEARCH_FIRST_PROMPT` 生成一个初步计划。
2.  **执行与产出**: 系统执行这个初步计划，产出第一批压缩后的信息摘要。
3.  **第二次迭代**: `generate_search_plan` 再次被调用，但这次，**第一次迭代的计划和产出的摘要**会被填入 `DEEPRESEARCH_NEXT_PROMPT` 模板中，作为上下文信息提供给“规划师”LLM。
4.  **进化**: “规划师”LLM看到上次的行动和结果后，就能做出更明智的判断，生成一个更深入或修正过的“下一轮计划”。

**代码逻辑展示：**
```python
# app/search/fc_deepresearch.py

def deepresearch_tool(messages: list[dict]):
    # ... (初始化)

    # 生成第一个计划 (无记忆)
    current_search_plan_steps = generate_search_plan(
        messages=messages,
        web_reference=...
    )

    # ... (执行第一个计划，产出 accumulated_search_results)

    # 进入迭代循环
    while len(executed_search_plans) < max_plan_iterations:

        # 生成后续计划 (核心的“记忆”构建环节)
        current_search_plan_steps = generate_search_plan(
            messages=messages,
            # 将之前的计划和结果作为“记忆”传入
            previous_plan=str([plan.get("search_purpose",'') for plan in executed_search_plans]),
            previous_search_results=accumulated_search_results.to_str(),
            max_remaining_steps=...
        )

        # ... (执行新计划，并更新 accumulated_search_results)
```

---

## 第四部分：端到端完整案例演示

我们以 `README.md` 中提到的冷门问题为例：“**rk3399 怎么在linux下,使用qemu 的kvm加速**”，来模拟一次完整的端到端流程。

### 1. 案例数据样本（模拟）
假设搜索引擎API (`search_api_worker`) 返回了以下3个初步结果：

*   **网页1**: { "title": "QEMU on RK3399 - General - Arm Community", "content": "Running QEMU with KVM acceleration on RK3399 boards...", "url": "https://community.arm.com/..." }
*   **网页2**: { "title": "在RK3399上编译Android 10系统并运行 - CSDN博客", "content": "本文记录了在RK3399开发板上编译安卓系统的完整过程...", "url": "https://blog.csdn.net/..." }
*   **网页3**: { "title": "[RFC PATCH] arm64: dts: rockchip: Add GIC CPU interface nodes for rk3399", "content": "This patch adds the GIC CPU interface nodes needed for KVM on rk3399...", "url": "https://lore.kernel.org/..." }

### 2. 分步操作演示

#### **阶段一：触发与初步规划**
1.  **API调用**: 用户向 `/v1/chat/completions` 发送请求，`model` 字段为 `...-deep-research`，`messages` 包含 "rk3399...kvm加速"。
2.  **生成计划**: `fc_deepresearch.py` 中的 `generate_search_plan` 被调用。其`prompt`大致内容为：
    > **Prompt to Planner Agent:**
    > "根据用户问题‘rk3399...kvm加速’，制定一个搜索计划，包含搜索关键词和目的。"
3.  **LLM输出 (规划师)**: (模拟JSON输出)
    ```json
    {
      "steps": [
        {
          "search_purpose": "查找在RK3399上使用QEMU并启用KVM加速的具体步骤和关键配置。",
          "query_keys": [
            {"key": "RK3399 QEMU KVM acceleration tutorial", "language": "en"},
            {"key": "RK3399 KVM GIC CPU interface", "language": "en"}
          ]
        }
      ]
    }
    ```

#### **阶段二：执行、评估与筛选**
1.  **执行搜索**: 系统使用上述关键词进行搜索，得到我们模拟的3个网页结果。
2.  **评估价值**: `search_after_ai.py` 中的 `evaluate_relevance` 被调用。
    > **Prompt to Evaluator Agent:**
    > "你的任务是‘查找RK3399上QEMU启用KVM的步骤’。下面是3个搜索结果，请为每个结果的相关性打分（1-10分）...
    > 索引 0: ...Arm Community...
    > 索引 1: ...CSDN编译安卓...
    > 索引 2: ...kernel.org GIC patch..."
3.  **LLM输出 (评估师)**: (模拟JSON输出)
    ```json
    { "0": 9, "1": 2, "2": 8 }
    ```
4.  **筛选结果**: 系统发现网页1和网页3得分最高，而网页2（编译安卓）相关性低，被丢弃。

#### **阶段三：抓取、压缩与信息提炼**
1.  **深度扫描**: `deepscan` 函数被调用，处理网页1和网页3。
2.  **内容压缩**: `compress_url_content` 被并行调用两次。以网页1为例：
    > **Prompt to Compressor Agent:**
    > "用户想知道‘RK3399上QEMU启用KVM的步骤’。这是从 `community.arm.com` 抓取的全文：[...网页1的全部Markdown内容...]。请提取与用户问题最相关的核心信息。"
3.  **LLM输出 (压缩师)**: (模拟输出)
    > **网页1压缩摘要**: "在RK3399上使用QEMU并启用KVM的关键在于内核必须正确配置GICv3。此外，由于RK3399是大小核架构（big.LITTLE），启动虚拟机时必须使用`taskset`命令将QEMU进程绑定到大核心（A72）上，否则会导致性能问题或启动失败。"

#### **阶段四：迭代与最终报告**
1.  **判断与再规划**: `deepresearch_tool` 的主循环判断当前信息可能还不够详尽。它将调用 `generate_search_plan` 进行第二轮规划。
    > **Prompt to Planner Agent (Iteration 2):**
    > "我们正在研究‘RK3399 KVM’。上一轮的计划是[...], 已发现关键信息：‘需要配置GICv3并使用taskset绑核’。请制定下一步的搜索计划，以获取更详细的配置代码或命令示例。"
2.  **(后续循环... 直至结束)**
3.  **最终总结**: 所有轮次收集到的压缩摘要（如上面那段）被汇总，发送给“总结师”LLM。
    > **Prompt to Summarizer Agent:**
    > "根据以下核心信息，生成一份关于‘如何在RK3399上使用QEMU并启用KVM加速’的详细指南...
    > 信息1: ...GICv3和taskset绑核是关键...
    > 信息2: ...具体的qemu-system-aarch64启动命令示例...
    > 信息3: ...检查KVM是否启用的命令是/dev/kvm..."
4.  **最终输出**: 生成一份结构化的完整报告，呈现给用户。
