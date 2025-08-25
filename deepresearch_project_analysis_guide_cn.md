# DeepResearch项目数据处理工作流深度解析（最终增强版）

---

## 第一部分：核心思想与高级工作流

### 1.1 引言

本文档是根据您的反馈和要求，对 **DeepResearch** 项目内部工作流程进行的最终版深度解析。本文的目标是**极其详尽地**剖析该项目处理海量网络数据的完整流程，特别是其**数据在每一步是如何被转换、提炼和传递的**，以便您能从中学习其设计精髓，为处理您自己的业务数据（如日志、数据库查询结果等）提供思路。

### 1.2 高级工作流：基于多智能体的数据蒸馏管道

DeepResearch 的核心是一个**基于多智能体（Multi-Agent）的数据蒸馏管道（Data Distillation Pipeline）**。它并非一次性将原始数据抛给一个通用大模型，而是通过一个精密的、可迭代的流程，让不同角色的AI智能体协同工作，逐步将海量的、低信息密度的原始网页，蒸馏成小体积、高信息密度的、与用户查询高度相关的知识片段。

**工作流闭环（代码实现于 `fc_deepresearch.py` 中的 `deepresearch_tool` 函数）:**

1.  **规划 (Planning)**
    *   **执行者**: `Planner Agent` (由 `generate_search_plan` 函数实现)。
    *   **任务**: 分析用户初始请求，生成结构化的研究计划。在后续迭代中，根据已知结果动态调整计划。
2.  **执行 (Execution)**
    *   **搜索 (Search)**: 调用外部搜索引擎（如 SearXNG），获取原始URL列表。
    *   **评估 (Evaluate)**:
        *   **执行者**: `Evaluator Agent` (由 `evaluate_relevance` 函数实现)。
        *   **任务**: 对搜索到的URL进行快速评估和打分，过滤掉明显不相关的结果，避免在无用数据上浪费资源。
    *   **蒸馏 (Distill)**:
        *   **执行者**: `Compressor Agent` (由 `compress_url_content` 函数实现)。
        *   **任务**: 对评估通过的高价值URL进行“查询感知型压缩”，即根据用户的原始问题，提取网页核心信息，生成精炼摘要。这是数据处理的核心。
3.  **迭代 (Iteration)**
    *   **执行者**: `deepresearch_tool` 主循环。
    *   **任务**: 将本轮蒸馏出的知识（精炼摘要）作为“记忆”，反馈给 `Planner Agent`，启动下一轮更精准的“规划->执行”循环。
4.  **总结 (Summarization)**
    *   **执行者**: `Summarizer Agent` (在 `process_messages` 中调用)。
    *   **任务**: 整合所有迭代轮次中收集到的知识片段，生成最终的、结构化的研究报告。

---

## 第二部分：关键模块与提示词工程深度分析

### 2.1 规划师 (Planner) 的大脑：`generate_search_plan` 与提示词

此函数位于 `app/search/fc_deepresearch.py`。其核心是两个关键的提示词（Prompt），定义在 `app/utils/prompt.py` 中。

*   **`DEEPRESEARCH_FIRST_PROMPT`**: 用于第一次迭代，它告诉LLM要从零开始，根据用户问题制定一个全面的搜索计划。
*   **`DEEPRESEARCH_NEXT_PROMPT`**: 用于后续迭代，这是实现“记忆”和“进化”的关键。
    > **Prompt 节选 (`DEEPRESEARCH_NEXT_PROMPT`)**:
    > `...你之前的搜索计划是: {previous_search_plan}。根据这些计划，你已经获得了以下结果: {previous_search_results}。请注意，这些是你已经探索过的领域，避免重复。基于以上信息，请为下一步生成一个补充性的、更深入的搜索计划...`
    *   **深度解析**: 这个Prompt通过 `{previous_search_plan}` 和 `{previous_search_results}` 这两个占位符，将上一轮的行动和结果明确地告知LLM，使其能够进行自我评估和修正，从而避免重复劳动，并向更深或更广的方向探索。

### 2.2 评估师 (Evaluator) 的火眼金睛：`evaluate_relevance`

此函数位于 `app/search/search_after_ai.py`。它在进行昂贵的网页抓取和压缩**之前**，先用一个低成本的LLM调用来快速过滤。

> **Prompt (`RELEVANCE_EVALUATION_PROMPT`)**:
> `...你的任务是判断以下搜索结果与搜索目标“{search_purpose}”的相关性。请为每个结果打分（1-10）。..`
*   **深度解析**: 这一步是至关重要的**成本控制器**和**质量过滤器**。它确保了只有最可能相关的网页才会被送入下一阶段，极大地节省了后续爬虫和压缩模型的API调用成本，并从源头上提高了最终报告的信噪比。

### 2.3 压缩师 (Compressor) 的蒸馏技术：`compress_url_content`

此函数位于 `app/utils/compress_content.py`。这是数据处理的核心，实现了“查询感知型压缩”。

> **Prompt (构建于 `by_openai` / `by_gemini` 函数内)**:
> `用户需要的信息:{user_input}\n\n 网页的标题与摘要是{title} 网页的url是{url} 网页内容:{html_content}`
*   **深度解析**: 这个Prompt的设计是整个项目的点睛之笔。它没有让LLM做一个通用的、无目的的网页摘要，而是强制其在压缩内容时，必须时刻以 `{user_input}` (用户的原始问题) 为准绳。这使得压缩过程变成了一个**有监督的、高效率的信息提取过程**，产出的不再是泛泛的摘要，而是可以直接用于回答用户问题的“知识片段”。

---

## 第三部分：端到端数据流演示

我们以 `README.md` 中的冷门问题为例：“**rk3399 怎么在linux下,使用qemu 的kvm加速**”，来极其详细地追踪数据在管道中的每一步转换。

### **数据流起点：用户输入**
*   **数据**: `{"role": "user", "content": "rk3399 怎么在linux下,使用qemu 的kvm加速"}`
*   **触发**: API路由检测到模型名称含 `deep-research`，调用 `deepresearch_tool`。

---

### **第一轮迭代**

#### **Step 1: 规划 (Planning)**
*   **输入**: 用户问题。
*   **动作**: `generate_search_plan` 函数调用 `SEARCH_KEYWORD_MODEL`。
*   **Prompt**: `DEEPRESEARCH_FIRST_PROMPT` 被填充。
*   **输出 (数据转换)**: 从自然语言问题变为结构化JSON。
    ```json
    {
      "steps": [
        {
          "search_purpose": "查找在RK3399芯片的Linux环境下，使用QEMU并开启KVM硬件加速的具体方法、步骤和依赖。",
          "query_keys": [
            {"key": "RK3399 QEMU KVM acceleration tutorial", "language": "en"},
            {"key": "qemu-system-aarch64 KVM rk3399", "language": "en"}
          ]
        }
      ]
    }
    ```

#### **Step 2: 搜索 (Searching)**
*   **输入**: 上一步生成的JSON中的 `query_keys`。
*   **动作**: `search_api_worker` 并发调用搜索引擎API。
*   **输出 (数据转换)**: 从搜索关键词变为一个充满噪声的、包含多个网页信息的JSON列表。
    ```json
    [
      { "title": "QEMU on RK3399 - General - Arm Community", "content": "Running QEMU with KVM...", "url": "https://community.arm.com/f/a"},
      { "title": "在RK3399上编译Android 10 - CSDN", "content": "本文记录了在RK3399开发板上编译安卓...", "url": "https://blog.csdn.net/b"},
      { "title": "[RFC PATCH] arm64: dts: rockchip: Add GIC CPU nodes", "content": "This patch adds GIC nodes needed for KVM...", "url": "https://lore.kernel.org/c"}
    ]
    ```

#### **Step 3: 评估 (Evaluating)**
*   **输入**: 上一步生成的JSON列表和第一步生成的`search_purpose`。
*   **动作**: `evaluate_relevance` 函数调用 `EVALUATE_MODEL`。
*   **Prompt**: `RELEVANCE_EVALUATION_PROMPT` 被填充，要求对每个网页打分。
*   **输出 (数据转换)**: 从JSON列表变为一个只包含高分结果的、被过滤后的JSON列表。
    ```json
    [
      { "title": "QEMU on RK3399 - General - Arm Community", "content": "...", "url": "https://community.arm.com/f/a", "relevance_score": 9},
      { "title": "[RFC PATCH] arm64: dts: rockchip: Add GIC CPU nodes", "content": "...", "url": "https://lore.kernel.org/c", "relevance_score": 8}
    ]
    ```
    *(CSDN编译安卓的文章因得分低被丢弃)*

#### **Step 4: 蒸馏 (Distilling)**
*   **输入**: 上一步过滤后的高分URL列表。
*   **动作**: `deepscan` 函数并发调用 `compress_url_content`。
*   **Prompt**: 对每个URL，填充其抓取到的全文内容和用户原始问题，调用 `COMPRESS_MODEL`。
*   **输出 (数据转换)**: 将每个URL的全文内容，蒸馏成一段高信息密度的“知识片段”。
    *   **知识片段1 (源自 URL a)**: `"在RK3399上使用QEMU并启用KVM的关键在于内核必须正确配置GICv3。此外，由于是大小核架构，启动虚拟机时必须使用`taskset`命令将QEMU进程绑定到大核心（A72）上，否则会导致启动失败。"`
    *   **知识片段2 (源自 URL c)**: `"内核设备树（DTS）中需要为rk3399添加GIC CPU接口节点，这是支持KVM虚拟化的前提。相关的内核补丁（Patch）已经提交至社区。"`

---

### **第二轮迭代**

#### **Step 5: 再规划 (Re-Planning)**
*   **输入**: 用户原始问题 + **第一轮迭代产出的所有知识片段和执行过的计划**。
*   **动作**: `generate_search_plan` 再次被调用，但这次使用 `DEEPRESEARCH_NEXT_PROMPT`。
*   **Prompt**: `DEEPRESEARCH_NEXT_PROMPT` 被填充，内容大致为：“我们正在研究RK3399的KVM问题，已经知道了‘需要配置GICv3’和‘需要用taskset绑核’。请制定下一步计划，帮我找到具体的taskset命令用法或者QEMU启动命令示例。”
*   **输出 (数据转换)**: 生成一个更具针对性的新研究计划。
    ```json
    {
      "steps": [
        {
          "search_purpose": "寻找在RK3399上配合QEMU使用的`taskset`命令的具体语法和QEMU启动脚本示例。",
          "query_keys": [
            {"key": "qemu rk3399 taskset command example", "language": "en"},
            {"key": "qemu-system-aarch64 rk3399 startup script", "language": "en"}
          ]
        }
      ]
    }
    ```

*后续步骤会重复 **Step 2, 3, 4**，但使用的是这个更精确的研究计划，从而获取更深层次的信息。*

---

### **最终步骤：总结**

*   **输入**: 所有迭代轮次中收集到的全部“知识片段”。
*   **动作**: `process_messages` 调用 `SUMMARY_MODEL`。
*   **Prompt**: “请根据以下核心信息，为‘如何在rk3399上使用qemu并启用kvm加速’这个问题，生成一份详尽的、步骤清晰的指南...”
*   **输出 (最终报告)**: 一份整合了所有知识、结构清晰的最终答案。

---

## 第四部分：可借鉴的核心思想（拓展思路）

通过对DeepResearch项目的深度分析，您可以将以下核心思想应用到您自己的业务场景中：

1.  **日志分析场景**:
    *   **不要直接分析原始日志**。先用一个廉价、快速的LLM调用（或Grep/AWK等传统工具）做**粗过滤**（如只保留含`ERROR`或`500`的行），将数据量减少99%。
    *   对过滤后的少量数据，再使用更高阶的LLM进行**根本原因分析（RCA）**、**模式识别**或**趋势总结**。

2.  **海量数据库查询结果分析**:
    *   **不要直接处理所有行**。先让LLM分析**表结构（Schema）和少量样本数据**，让它告诉你哪些列与你的分析目标最相关。
    *   **分块处理**。对每一块数据，只提取LLM认为最相关的那几列信息，并让其进行**初步的、结构化的信息提取**（例如，将多行用户行为转换成单行的“用户路径”字符串）。
    *   **最后汇总分析**。将所有数据块产出的结构化信息汇总起来，再进行更高维度的统计和分析。

这个“**粗过滤 -> 精加工 -> 再汇总**”的管道式处理思想，是应对一切超出LLM上下文窗口的大数据分析任务的通用钥匙。
