---
name: seekdb-hybrid-search
description: 用 seekdb（MySQL 协议）为本地资料建立"向量 + 全文"混合检索：建表、写入片段与向量、用一条 SQL 完成语义与关键词融合排序。适用于给 Agent 增加本地知识库、论文/笔记检索、RAG 的召回层。
tags: [seekdb, oceanbase, vector-search, fulltext, rag, hybrid-retrieval]
---

# seekdb 混合检索助手

## 用途

把一批文本片段（笔记、文档、FAQ、代码注释）存进 seekdb，并提供**一条 SQL 同时完成语义与关键词检索**的能力，适合作为 RAG 的召回层或 Agent 的本地知识库。

## 前置条件

- 一个可连接的 seekdb 实例（默认 `127.0.0.1:2883`，用户 `root`，MySQL 协议）
- 片段向量由调用方的 embedding 模型生成；**建库与查询必须使用同一个模型与维度**

## 表结构

```sql
CREATE DATABASE IF NOT EXISTS kb DEFAULT CHARSET = utf8mb4;
USE kb;

CREATE TABLE IF NOT EXISTS chunks (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  doc_id VARCHAR(160) NOT NULL,
  doc_title VARCHAR(255) NOT NULL DEFAULT '',
  chunk_no INT NOT NULL DEFAULT 0,
  content TEXT NOT NULL,
  embedding VECTOR(512) NOT NULL,          -- 维度与 embedding 模型一致
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FULLTEXT INDEX ft_content(content) WITH PARSER ik,
  KEY idx_doc (doc_id)
) DEFAULT CHARSET = utf8mb4;
```

> seekdb 目前不允许同一张表同时建立向量索引与全文索引；片段数量在万级以下时直接全表
> `cosine_distance` 排序即可（实测 26 片段约 3 ms）。规模更大时拆成向量表与全文表，
> 用同一条 SQL 的 JOIN 融合。

## 写入

1. 按语义边界切分（建议 300 字、重叠 50），保留 `doc_id / doc_title / chunk_no`；
2. 批量写入时用事务，向量以 `[0.12,0.34,...]` 文本传入；
3. 提交后用**新连接**查一次 `SELECT COUNT(*) FROM chunks WHERE doc_id = ?`，确认索引刷新后可见。

## 检索（单条 SQL）

```sql
SELECT doc_title, chunk_no, content,
       cosine_distance(embedding, [查询向量]) AS vdist,
       MATCH(content) AGAINST('关键词') AS fts,
       0.7 * (1 - cosine_distance(embedding, [查询向量]))
     + 0.3 * LEAST(MATCH(content) AGAINST('关键词')
                   / (MATCH(content) AGAINST('关键词') + 5), 1) AS score
  FROM chunks
 WHERE MATCH(content) AGAINST('关键词') > 0
 ORDER BY score DESC
 LIMIT 6;
```

- 「仅语义」：去掉 `WHERE MATCH`，`ORDER BY cosine_distance(...)`；
- 「仅关键词」：`WHERE MATCH(...) > 0 ORDER BY fts DESC`；
- 限定范围：加 `AND doc_id = '...'`；
- 关键词从用户问题中抽取（2–4 个实词），embedding 用原问题。

## 回答要求

1. 结果按 `score` 降序，附出处（`doc_title` + 第几段）与原文片段；
2. 不要改写出处与原文；引用时保留原始措辞；
3. 无命中时如实说明"库中没有相关内容"，可建议放宽关键词或补充资料，不要编造；
4. 若种子数据是示例数据，需在回答中标注来源。

<!-- OB-S:0by3tHFpJazQjpbSmBAdXg -->
