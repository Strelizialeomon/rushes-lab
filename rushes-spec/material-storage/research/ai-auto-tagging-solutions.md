# AI 素材自动打标 / 可搜索属性 — 技术方案调研

**TL;DR**:自托管离线场景下的主流做法是「独立 ML 服务 + 三层识别能力(轻量打标模型 / 多模态大模型 VLM / 语义向量) + 混合检索」,全部组件有成熟开源实现,单张消费级显卡(10GB 级)即可起步。本笔记是 [`../ai-tagging-base.md`](../ai-tagging-base.md) 的依据文档。

调研日期:2026-09-15

## 1. 打标 / 属性识别路线对比

| 路线 | 代表 | 长处 | 短板 |
| --- | --- | --- | --- |
| 专用打标模型(分类头) | [RAM++](https://github.com/xinyu1205/recognize-anything)(开放集,英文标签)、Florence-2、[WD14/SmilingWolf 系](https://github.com/corkborg/wd14-tagger-standalone)(booru 词表) | 毫秒级、ONNX 可 CPU 跑、标签确定性强、可批量 | 语义浅(只有"是什么",没有"在干嘛/什么风格");英文词表为主;医美垂直词表需微调 |
| 多模态大模型(VLM)生成式 | Qwen2.5-VL / Qwen3-VL(7B/3B)、InternVL、MiniCPM-V | 中文最强;可输出结构化 JSON(标签+场景+颜色+图中文字 OCR+描述);[支持 zero-shot 检测与结构化抽取](https://blog.roboflow.com/qwen2-5-vl-zero-shot-object-detection/) | 7B 需 ~6GB 显存(4bit 量化);秒级/图;有幻觉风险 → 需 guided JSON + 置信度 + 独立标签区隔离 |
| 云端打标 API | 各厂多模态 API | 零运维 | **一票否决**:生产要求可完全离线(ops-manual.md:602),且医美素材隐私不能出内网 |

**结论**:VLM(Qwen 系)为主力,专用打标模型做兜底/快速通道;两者标签都进独立 AI 标签区,与人工标签隔离。

## 2. 语义检索(以文搜图)

| 候选 | 说明 |
| --- | --- |
| SigLIP2 / Chinese-CLIP | 轻量(1–2GB)、成熟;中文 query 首选 multilingual/中文变体 |
| [Qwen3-VL-Embedding](https://arxiv.org/pdf/2607.15689) | 新一代,帧检索/图文检索 benchmark 领先,2B 级显存 ~2GB |

向量与关键词做**混合检索**(向量粗召回 → 元数据/权限过滤 → 融合排序)是业界标准漏斗([参考](https://www.deepdata.cn/anarticle_idBd0jBL--.html));注意**模型版本与向量绑定**——换模型旧向量即失效,需全量重灌。

## 3. 向量库选型

| 候选 | 适用 |
| --- | --- |
| **pgvector**(选定) | 已有 PG16,零新增组件、备份一体;HNSW 索引在 ~1M 向量内性能与专用库同档([对比](https://www.matthewswong.com/en/blog/pgvector-vs-pinecone-comparison/))。我们的量级(万~十万素材)远在舒适区 |
| Qdrant / Milvus | 百万级以上、或需要独立扩缩容时再评估([2025 benchmark](https://inductivee.com/blog/vector-database-performance-benchmarks-2025)) |

## 4. 模型服务(serving)

| 候选 | 定位 |
| --- | --- |
| **vLLM**(选定,VLM) | 生产级:连续批处理、OpenAI 兼容 API、guided JSON 输出;[对比测试吞吐/成功率显著优于 Ollama](https://www.mdpi.com/2076-3417/16/11/5435) |
| ONNX Runtime(选定,小模型) | RAM++/WD14/SigLIP 等小模型的标准推理路径,可 CPU 兜底 |
| Ollama | 上手快但偏单机玩具,生产 serving 不推荐([对比](https://pub.towardsai.net/ollama-vs-vllm-which-open-source-inference-stack-should-you-actually-use-b9812cfbb175)) |

## 5. 参考系统

- **[Immich](https://deepwiki.com/immich-app/immich)**(自托管相册,AGPL):最贴近的样板——ML 独立容器、任务队列驱动、CLIP 语义搜索 + InsightFace 人脸、模型离线加载。我们借其架构形态(ML 微服务 + 队列),不引代码。
- PhotoPrism:TensorFlow 分类打标,思路同上代,不展开。

## 6. 视频路线(本底座预留,一期不做)

抽帧(现有 ffmpeg 链路,workers/main.py:500-522)→ 帧走图片管线 → 标签聚合;语音轨 Whisper(faster-whisper)转写进可搜索文本。人脸/人物聚类(InsightFace,Immich 同款)医美场景敏感,后期单独评估。
