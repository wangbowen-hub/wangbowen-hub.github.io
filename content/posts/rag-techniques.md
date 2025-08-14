---
title: "Rag Techniques"
date: 2025-07-22T21:59:11+08:00
# bookComments: false
# bookSearchExclude: false
---

Reference:https://github.com/NirDiamant/RAG_Techniques/tree/main

# RAG Techniques

##  Foundational

### Proposition Chunking

将原始文档分块后，根据每个分块生成命题（事实性、自包含的陈述句），通过另一个evaluator LLM 通过从准确性、清晰度、完整性和简洁性四个维度对命题进行评分，从而评估命题质量，通过质量检查的命题将被保留，嵌入到向量存储中，这使得在进行查询时能够基于命题检索答案。

原始文档向量存储和命题向量存储都进行保留和可供检索，这两种检索方式各有优缺点，命题检索适合精准细节回答，原始文档检索更适合需要结合上下文信息综合理解的查询：

![image-20250722221020102](/Users/bowenwang/Library/Application Support/typora-user-images/image-20250722221020102.png)

## Query Enhancement

### Query Transformations

* 查询重写（Query Rewriting）Reformulates queries to be more specific and detailed.

* 回退式提问（Step-back Prompting）Generates broader queries for better context retrieval.

* 子查询分解（Sub-query Decomposition）Breaks down complex queries into simpler sub-queries.

  ```python
  #############查询重写（Query Rewriting）##################
  re_write_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
  
  # Create a prompt template for query rewriting
  query_rewrite_template = """You are an AI assistant tasked with reformulating user queries to improve retrieval in a RAG system. 
  Given the original query, rewrite it to be more specific, detailed, and likely to retrieve relevant information.
  
  Original query: {original_query}
  
  Rewritten query:"""
  
  query_rewrite_prompt = PromptTemplate(
      input_variables=["original_query"],
      template=query_rewrite_template
  )
  
  # Create an LLMChain for query rewriting
  query_rewriter = query_rewrite_prompt | re_write_llm
  
  def rewrite_query(original_query):
      """
      Rewrite the original query to improve retrieval.
      
      Args:
      original_query (str): The original user query
      
      Returns:
      str: The rewritten query
      """
      response = query_rewriter.invoke(original_query)
      return response.content
  ```

  ```python
  #############回退式提问（Step-back Prompting）##################
  step_back_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
  
  
  # Create a prompt template for step-back prompting
  step_back_template = """You are an AI assistant tasked with generating broader, more general queries to improve context retrieval in a RAG system.
  Given the original query, generate a step-back query that is more general and can help retrieve relevant background information.
  
  Original query: {original_query}
  
  Step-back query:"""
  
  step_back_prompt = PromptTemplate(
      input_variables=["original_query"],
      template=step_back_template
  )
  
  # Create an LLMChain for step-back prompting
  step_back_chain = step_back_prompt | step_back_llm
  
  def generate_step_back_query(original_query):
      """
      Generate a step-back query to retrieve broader context.
      
      Args:
      original_query (str): The original user query
      
      Returns:
      str: The step-back query
      """
      response = step_back_chain.invoke(original_query)
      return response.content
  ```

  ```python
  #############子查询分解（Sub-query Decomposition）##################
  sub_query_llm = ChatOpenAI(temperature=0, model_name="gpt-4o", max_tokens=4000)
  
  # Create a prompt template for sub-query decomposition
  subquery_decomposition_template = """You are an AI assistant tasked with breaking down complex queries into simpler sub-queries for a RAG system.
  Given the original query, decompose it into 2-4 simpler sub-queries that, when answered together, would provide a comprehensive response to the original query.
  
  Original query: {original_query}
  
  example: What are the impacts of climate change on the environment?
  
  Sub-queries:
  1. What are the impacts of climate change on biodiversity?
  2. How does climate change affect the oceans?
  3. What are the effects of climate change on agriculture?
  4. What are the impacts of climate change on human health?"""
  
  
  subquery_decomposition_prompt = PromptTemplate(
      input_variables=["original_query"],
      template=subquery_decomposition_template
  )
  
  # Create an LLMChain for sub-query decomposition
  subquery_decomposer_chain = subquery_decomposition_prompt | sub_query_llm
  
  def decompose_query(original_query: str):
      """
      Decompose the original query into simpler sub-queries.
      
      Args:
      original_query (str): The original complex query
      
      Returns:
      List[str]: A list of simpler sub-queries
      """
      response = subquery_decomposer_chain.invoke(original_query).content
      sub_queries = [q.strip() for q in response.split('\n') if q.strip() and not q.strip().startswith('Sub-queries:')]
      return sub_queries
  ```

  

### HyDE (Hypothetical Document Embedding)

传统检索方法常常难以处理简短query与较长、更详细文档之间的语义鸿沟。HyDE 通过将query扩充成完整的假设性文档来解决这个问题，通过使查询表示更接近向量空间中的文档表示，从而可能提高检索相关性。

![HyDe](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/HyDe.svg?raw=1)

```python
class HyDERetriever:
    def __init__(self, files_path, chunk_size=500, chunk_overlap=100):
        self.llm = ChatOpenAI(temperature=0, model_name="gpt-4o-mini", max_tokens=4000)

        self.embeddings = OpenAIEmbeddings()
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.vectorstore = encode_pdf(files_path, chunk_size=self.chunk_size, chunk_overlap=self.chunk_overlap)
    
        
        self.hyde_prompt = PromptTemplate(
            input_variables=["query", "chunk_size"],
            template="""Given the question '{query}', generate a hypothetical document that directly answers this question. The document should be detailed and in-depth.
            the document size has be exactly {chunk_size} characters.""",
        )
        self.hyde_chain = self.hyde_prompt | self.llm

    def generate_hypothetical_document(self, query):
        input_variables = {"query": query, "chunk_size": self.chunk_size}
        return self.hyde_chain.invoke(input_variables).content

    def retrieve(self, query, k=3):
        hypothetical_doc = self.generate_hypothetical_document(query)
        similar_docs = self.vectorstore.similarity_search(hypothetical_doc, k=k)
        return similar_docs, hypothetical_doc

```

### HyPE (Hypothetical Prompt Embedding)

* HyPE 不是直接嵌入原始文本块，而是为每个文本块生成多个假设性query，并将这些query进行embedding。将这这些预先计算的问题模拟了用户查询，提高了与现实世界搜索的匹配度。这种方法无需像 HyDE 需要在运行时生成假设性文档，检索速度与标准RAG相当。

* 每个假设性问题都会进行embedding，并将每个假设性问题嵌入向量与其原始文本块相关联，检索时是问题-假设性问题匹配，而非问题-文档匹配。

![HyPE](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/hype.svg?raw=1)



## Context Enrichment

 ### CCH（Contextual Chunk Headers）

* 在文本块头部添加“标题”来为文本块提供更高层次的上下文，这个“标题”可以只是文档标题，也可以是文档摘要或完整的章节和子章节标题层次结构。
* 如果在检索过程中使用reranker，也需要将标题和文本块文本拼接。
* 输入给LLM的上下文也同样需要拼接标题，这能为 LLM 提供更多上下文信息。

![Your Technique Name](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/contextual_chunk_headers.svg?raw=1)

### RSE（Relevant Segment Extraction）

从召回的文本块中再筛选一遍，筛选出更符合query的

![Relevant segment extraction](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/relevant-segment-extraction.svg?raw=1)



### Context Enrichment Window for Document Retrieval

为每个检索到的块添加相邻文本块，从而提升送给LLM上下文的连贯性和完整性。

![context enrichment window](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/vector-search-comparison_context_enrichment.svg?raw=1)

![context enrichment window](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/context_enrichment_window.svg?raw=1)

### Semantic Chunking

根据语义对文档进行chunk，而不是chunk size和chunk overlapping

![Self RAG](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/semantic_chunking_comparison.svg?raw=1)

### Contextual Compression

传统文档检索通常返回整个文档块或完整文档，其中可能包含无关信息。Contextual Compression通过LLM提取和压缩仅与检索最相关的文档部分，从而实现更聚焦、更高效的信息检索。

![contextual compression](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/contextual_compression.svg?raw=1)

### Document Augmentation through Question Generation for Enhanced Retrieval

根据文本块级别/文本片段生成问题，同文本片段一起embedding，但送入LLM的context还是文本块（文本片段是从文本块中切分出来的）

```python
文本块=split_document(原文档)
文本片段=split_document(文本块)
```

**Document Augmentation through Question Generation与HyPE的区别：**（chatgpt回答，Document Augmentation through Question Generation 这部分和colab代码对不上）

* **Document Augmentation through Question Generation (Doc‑QG)**：先用 LLM 替每段文本生成「用户可能会问的问题（可选连同答案）」，把这些 *明文问题* 直接附加回语料，再走常规检索。

* **HyPE (Hypothetical Prompt Embedding)**：同样用 LLM 生成假设性问题，但**只把这些问题转成向量后保存**，丢弃文字本身；检索时拿查询向量和「假设问题向量」比对。



## Advanced Retrieval

### Fusion Retrieval

将基于向量的相似性搜索与基于关键词的 BM25 检索相结合，旨在利用两种技术的优势，提升文档检索的整体质量和相关性。

![Fusion Retrieval](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/fusion_retrieval.svg?raw=1)

### Reranking

通过对初始检索结果进行重新评估和排序，确保将最相关的信息优先用于后续处理或呈现。

Reranking有一下两种方法：

* LLM方法

![rerank llm](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/rerank_llm.svg?raw=1)

* 交叉编码器Cross Encoder方法

![rerank cross encoder](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/rerank_cross_encoder.svg?raw=1)



###  Hierarchical Indices

采用两级编码机制：文档级摘要（document-level summaries）和细节文本块（detailed chunks）。该方法旨在通过摘要先识别相关文档段落，再深入挖掘这些段落中的具体细节，从而提高信息检索的效率和相关性。

![hierarchical_indices](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/hierarchical_indices.svg?raw=1)

###  Dartboard Retrieval

确保检索到的信息既相关又不冗余。在大型数据库中，文档可能重复相似内容，导致前 k 项检索结果存在冗余。结合相关性与多样性可获得更丰富的文档集合，缓解内容过度相似造成的"信息茧房"效应。

### Multi-modal RAG with Captioning

![Reliable-RAG](https://github.com/NirDiamant/RAG_Techniques/blob/main/images/multi_model_rag_with_captioning.svg?raw=1)

### Multi-faceted Filtering（无）

### Ensemble Retrieval（无）

