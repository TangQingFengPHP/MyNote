# RAG 从原理到实战：让大模型先查资料，再回答问题

大模型能写代码、做总结，也能回答很多常识问题。但它通常不知道刚更新的内部制度、某个产品的私有参数，或者一份刚上传的 PDF 写了什么。把问题直接交给模型，答案可能听起来很顺，却没有资料依据。

RAG（Retrieval-Augmented Generation，检索增强生成）解决的是“回答前先查资料”这件事：先从指定知识库找出相关片段，再把问题和片段交给大模型组织答案。答案因此可以贴近资料，也能带上出处。

可以把它想成开卷问答：检索负责翻到相关页，模型负责读懂材料并整理成回答。资料没有写到的内容，系统应该说明依据不足，而不是让模型猜。

## 先看一遍完整流程

RAG 通常分为建库和问答两条流程。

```text
建库：文档 → 解析 → 切片 → 向量化 → 保存
问答：问题 → 向量化 → 检索相关片段 → 组织上下文 → 大模型回答 → 返回出处
```

建库通常在资料上传或更新时运行；问答流程则在每次提问时运行。向量数据库只是其中一个存储和检索组件，RAG 本身是一套从文档处理到答案生成的流程。用全文检索或数据库查询找到资料，再交给模型生成，也属于 RAG。

## 为什么要先切片

假设一本员工手册有两百页。每次提问都把整本手册塞给模型，会浪费上下文空间，也让模型更难聚焦。建库时会把长文档切成较小的文本块，通常叫 Chunk。

```text
员工手册
  ├── 第 1 片：年假申请条件
  ├── 第 2 片：年假天数与计算方式
  └── 第 3 片：病假申请流程
```

切片太大，检索结果会带进许多无关内容；切片太小，条件和结论可能被拆开。常见做法是优先按标题、段落和列表切分，再对超长内容按长度切开，并给相邻片段留少量重叠文本，避免重要句子刚好落在边界两侧。

每个片段还应该保存来源信息，例如文件名、页码、标题路径、文档版本和片段编号。只有文本而没有来源，回答就很难准确标注出处。

## 向量检索是怎么找到近似内容的

Embedding 模型会把文本转换成一组数字，称为向量。意思接近的文本，向量通常也更接近。

例如下面两句话用词不同，但含义相似：

```text
员工每年可以休几天年假？
年休假天数是多少？
```

向量检索可以根据语义找到相关文本，不要求问题和文档使用完全相同的词。检索时一般会把问题也转换成向量，再与知识库中的片段向量计算相似度，取前几条候选结果。

向量检索也有边界。订单号、错误码、产品型号这类精确词，关键词搜索往往更稳；权限、日期、部门等条件应当通过结构化过滤处理。实际系统经常把关键词检索与向量检索结合，再对候选结果重排。

## 一次问答具体发生什么

用户问：“入职两年有几天年假？”

知识库中有一段制度原文：

```text
员工连续工作满一年不满十年的，每年享有 5 天带薪年休假。
```

系统大致会这样处理：

1. 把问题转换成检索向量。
2. 从知识库取回与问题相关的片段。
3. 检查片段是否在当前用户可访问的范围内。
4. 把问题、相关片段和回答规则一并交给大模型。
5. 返回答案及片段来源。

生成环节可以要求模型只根据给定资料作答，并在证据不足时说明缺少信息。提示词能约束回答方式，但不能代替检索质量和权限校验。

## 本地跑一个可用的问答 Demo

下面用 Ollama 在本机运行模型，Python 标准库负责请求接口和计算余弦相似度，不需要安装向量数据库。示例会把文本片段和向量保存在内存中，适合看清 RAG 的关键步骤；程序退出后数据会消失。

### 1. 准备模型和文档

先安装并启动 Ollama，然后拉取一个 Embedding 模型和一个聊天模型：

```bash
ollama pull qwen3-embedding:0.6b
ollama pull qwen3:8b
```

模型大小和速度取决于运行设备。设备资源有限时，可以选择较小的聊天模型，并在代码里同步修改模型名称。

新建目录并放入 Markdown 或 TXT 文件：

```bash
mkdir -p rag-demo/docs
```

例如创建 `rag-demo/docs/员工休假制度.md`：

```markdown
# 员工休假制度

## 年休假
员工连续工作满一年不满十年的，每年享有 5 天带薪年休假。
连续工作满十年不满二十年的，每年享有 10 天带薪年休假。
年休假需要至少提前三个工作日提交申请，经直属负责人审批后生效。

## 病假
病假申请需要提交医疗机构出具的证明，并在返岗后补齐系统申请。
```

### 2. 编写问答程序

在 `rag-demo` 下新建 `rag_demo.py`：

```python
import json
import math
import urllib.request
from pathlib import Path


BASE_URL = "http://localhost:11434"
EMBED_MODEL = "qwen3-embedding:0.6b"
CHAT_MODEL = "qwen3:8b"
DOCS_DIR = Path(__file__).parent / "docs"


def ollama_post(path, payload):
    data = json.dumps(payload, ensure_ascii=False).encode("utf-8")
    request = urllib.request.Request(
        f"{BASE_URL}{path}",
        data=data,
        headers={"Content-Type": "application/json"},
    )
    with urllib.request.urlopen(request, timeout=300) as response:
        return json.loads(response.read().decode("utf-8"))


def embed(texts):
    result = ollama_post(
        "/api/embed",
        {"model": EMBED_MODEL, "input": texts},
    )
    return result["embeddings"]


def split_text(text, size=450, overlap=80):
    """按字符切分演示文本；正式项目应优先按标题和段落切分。"""
    text = text.strip()
    if not text:
        return []

    chunks = []
    start = 0
    while start < len(text):
        end = min(start + size, len(text))
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        if end == len(text):
            break
        start = end - overlap
    return chunks


def load_chunks():
    chunks = []
    for path in sorted(DOCS_DIR.glob("*.md")) + sorted(DOCS_DIR.glob("*.txt")):
        content = path.read_text(encoding="utf-8")
        for index, text in enumerate(split_text(content), start=1):
            chunks.append({
                "source": path.name,
                "chunk_id": index,
                "text": text,
            })
    if not chunks:
        raise RuntimeError(f"没有在 {DOCS_DIR} 找到 .md 或 .txt 文档")
    return chunks


def cosine_similarity(a, b):
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(y * y for y in b))
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return dot / (norm_a * norm_b)


def retrieve(question, chunks, vectors, top_k=3):
    question_vector = embed([question])[0]
    ranked = []
    for chunk, vector in zip(chunks, vectors):
        score = cosine_similarity(question_vector, vector)
        ranked.append((score, chunk))
    ranked.sort(key=lambda item: item[0], reverse=True)
    return ranked[:top_k]


def answer(question, hits):
    evidence = []
    for number, (score, chunk) in enumerate(hits, start=1):
        evidence.append(
            f"[资料{number}] 文件：{chunk['source']}，片段：{chunk['chunk_id']}，"
            f"相似度：{score:.3f}\n{chunk['text']}"
        )

    prompt = f"""请根据资料回答问题。
资料只能作为事实依据。资料没有提供答案时，明确说明“现有资料不足以回答”。
回答中的事实后标注对应资料编号，例如 [资料1]。不得补造文档中没有的规则。

问题：
{question}

资料：
{chr(10).join(evidence)}
"""
    result = ollama_post(
        "/api/chat",
        {
            "model": CHAT_MODEL,
            "messages": [{"role": "user", "content": prompt}],
            "stream": False,
        },
    )
    return result["message"]["content"]


def main():
    chunks = load_chunks()
    print(f"读取到 {len(chunks)} 个文本片段，正在生成向量……")
    vectors = embed([chunk["text"] for chunk in chunks])

    while True:
        question = input("\n请输入问题（输入 q 退出）：").strip()
        if question.lower() == "q":
            break
        if not question:
            continue

        hits = retrieve(question, chunks, vectors)
        print("\n检索到的资料：")
        for number, (score, chunk) in enumerate(hits, start=1):
            print(f"[{number}] {chunk['source']} 第 {chunk['chunk_id']} 片，分数 {score:.3f}")
            print(f"    {chunk['text'][:160]}")

        print("\n回答：")
        print(answer(question, hits))


if __name__ == "__main__":
    main()
```

运行：

```bash
cd rag-demo
python3 rag_demo.py
```

输入：

```text
入职两年有几天年假？
```

回答可能类似：

```text
连续工作满一年不满十年的员工，每年享有 5 天带薪年休假。[资料1]
```

再问“年假申请要提前多久？”，答案应引用制度中的“三个工作日”。问“年终奖是多少？”时，演示提示词会要求模型说明资料不足。实际效果仍取决于模型是否遵守提示，生产系统还需要设置检索分数阈值和无结果处理。

### 3. 代码各部分做了什么

| 代码 | 对应环节 | 作用 |
| --- | --- | --- |
| `load_chunks`、`split_text` | 文档解析与切片 | 读取本地文件并拆成小段 |
| `embed` | 向量化 | 调用 Embedding 模型把文本转成向量 |
| `cosine_similarity` | 相似度计算 | 计算问题与文档片段的接近程度 |
| `retrieve` | 检索 | 按分数选出前几段资料 |
| `answer` | 生成 | 将证据与问题交给聊天模型，并要求标出来源 |

程序每次启动都会重新向量化。资料较多时，应将向量和片段元数据保存到向量数据库，例如 Milvus、Qdrant 或 pgvector；文档内容和权限等业务数据则保存在适合的存储中。

## 让 Demo 更接近实际项目

上面的程序是为了展示主流程，进入实际业务前通常需要补上这些能力：

### 混合检索

把向量检索和关键词检索并行执行。向量擅长找语义相近的内容，关键词检索擅长精确匹配型号、编号和专有名词。之后可以用 RRF 等方法融合排序，再按需使用 Reranker 对候选片段精排。

### 更可靠的切片

按标题和段落切分，保留页码、表格标题、文档路径等定位信息。中文字符数和模型 token 数并不相等；正式配置应基于实际 Embedding 模型的 tokenizer，并通过问答样本调整片段长度和重叠范围。

### 权限和版本

检索前按用户可访问的知识库、文档状态和版本做过滤。不能先取回所有资料，再只靠提示词让模型忽略无权访问的内容。文档更新时也要让旧版本及时退出检索结果。

### 引用校验与无答案处理

引用应绑定到本次检索返回的文档、版本和片段，避免模型编造出处。检索分数低、证据互相冲突或关键条件缺失时，应返回“资料不足”或提出需要补充的信息。

### 评测检索和答案

准备一批真实问题，标注正确资料片段和答案要点。分别观察“正确片段有没有被召回”“答案是否符合片段”“引用是否指向正确位置”。回答文字流畅不代表检索正确。

## RAG 和微调有什么区别

| 对比项 | RAG | 微调 |
| --- | --- | --- |
| 主要用途 | 提供可更新的外部资料 | 调整模型的行为、格式或特定任务能力 |
| 更新知识 | 更新文档并重新索引 | 通常需要准备数据并重新训练或微调 |
| 出处追溯 | 可以将答案关联到检索片段 | 模型本身不会自动提供可靠出处 |
| 私有知识问答 | 常见选择 | 资料频繁变化时维护成本较高 |

两者可以组合：用微调稳定回答格式，再用 RAG 提供最新业务资料。

## 常见问题

### RAG 一定要用向量数据库吗？

不一定。小型 Demo 可以在内存里计算相似度；数据量增加后再使用向量数据库。只用 Elasticsearch、数据库全文索引或其他检索方式，也可以把召回内容交给大模型完成 RAG。

### 加了 RAG，模型就不会出错了吗？

不会。检索可能找错片段，文档可能过期或不完整，模型也可能误读证据。RAG 能增加回答依据，不能自动保证答案正确。权限过滤、引用校验和实际问题评测都需要单独处理。

### 检索结果不准时先看哪里？

先检查原文是否正确解析，再看切片是否把问题和答案拆开，然后检查检索结果排名。若关键词精确匹配差，可以加入全文检索；若候选结果相关但顺序不理想，可以评估 Reranker；若证据已经正确而答案仍错误，再检查提示词和模型表现。

## 总结

RAG 的关键不在于接上某一个向量数据库，而在于把资料可靠地解析、切分、检索，再把有出处的证据交给模型。理解这条链路后，组件选择会容易很多：先用本地 Demo 跑通，再用真实问题检查召回、答案和引用，最后按数据规模补充向量库、混合检索、权限控制和评测。

Ollama 接口与模型资料：[Ollama Embeddings 文档](https://docs.ollama.com/capabilities/embeddings)、[Qwen3 Embedding 模型库](https://ollama.com/library/qwen3-embedding)。
