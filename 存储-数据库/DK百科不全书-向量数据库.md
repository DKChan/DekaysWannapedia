## 通用

### 常见数据库

- **索引类型：**
  - HNSW（图结构）：高精度召回（>95%）
  - IVF-PQ（聚类+量化）：高压缩比（内存占用降低8倍）
  - LSH（局部敏感哈希）：超高速检索
- **性能：** 10亿级向量检索<100ms
| 工具 | 核心优势 | 适用场景 | 部署复杂度 |
|:---|:---|:---|:---|
| Milvus | 开源全功能王者，支持多索引/多云架构 | 企业级生产系统（如京东推荐引擎） | ⭐⭐⭐（3星） |
| Pinecone | Serverless全托管，API即服务 | 初创公司快速接入（无需运维） | ⭐（1星） |
| Weaviate | 内置向量生成模块（多模型集成） | 无嵌入模型的轻量化场景 | ⭐⭐（2星） |
| Qdrant | Rust开发，极致性能（>100k QPS） | 高并发实时搜索（如TikTok视频推荐） | ⭐⭐（2星） |
| pgvector | PostgreSQL扩展，完美兼容SQL生态 | 现有PG业务升级向量需求 | ⭐（1星） |
| Elasticsearch | 融合全文检索+向量搜索（8.x+） | 混合查询场景（如商品多维度筛选） | ⭐⭐⭐（3星） |

性能指标参考（来源：ANN-Benchmarks）：
- 10M向量检索延迟：Qdrant（2ms）、Milvus（5ms）、Pinecone（8ms）
- 精度召回率：HNSW > IVF-PQ > LSH

### 索引结构

**基础:**

1.  Flat Index 全量索引

暴力搜索, 所有向量逐个对比(对比方法比如计算余弦相似度); 100%精度, 但性能慢, 小量级可以

2.  近似最近邻(ANN)

    a.  HNSW(Hierarchical Navigable Small World) 分层可导航小世界图: 多个分层, 查询时从顶到底导航, 顶层稀疏, 底层稠密

    b.  IVF(Inverted File Index) 倒排文件索引: Kmeans聚类, 向量分类后, 只在最近蔟搜索

    c.  PQ(Product Quantization) 产品量化/乘积量化:

    d.  ANNOY(spotify的)

**高级:**

1.  层次化索引: 多层次分层; 适用大规模文档库
2.  图索引: 以知识图谱存储实体关系; 复杂语义, 多跳遍历推理
3.  融合索引: 多种结合(如: IVF+)

**选取方案:**

中小规模+高精度: Flat Index 或者 HNSW

大规模+低资源: IVF / PQ

复杂语义查询: 图索引 / 层次化索引

通用场景: 融合索引(HNSW+BM25)

### 向量匹配算法

1.  基于特征

    a.  BM25

    b.  TF-IDF

    c.  jaccard相似度

2.  基于表征

    a.  词向量: word2vec, Glove

    b.  句向量: BERT, Sentence-Bert

3.  基于交互

    a.  Cross-Encoder, 利用transformer

    b.  ESIM(enhanced LSTM)

4.  近似最近邻(ANN)

    a.  HNSW

    b.  IVF-PQ

    c.  Annoy

距离度量方法

1.  余弦相似度
2.  欧氏距离
3.  内积, 点积, 点乘
4.  汉明距离

## Milvus

架构图

![descript](DK百科不全书-向量数据库-media/image3.png)
