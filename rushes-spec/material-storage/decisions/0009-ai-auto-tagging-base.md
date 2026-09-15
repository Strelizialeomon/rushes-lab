# ADR-0009:AI 打标底座 — 自托管 ML 微服务 + 独立 AI 标签区 + pgvector

- 状态: accepted
- 日期: 2026-09-15
- 关联: 方案全文 [../ai-tagging-base.md](../ai-tagging-base.md),调研 [../research/ai-auto-tagging-solutions.md](../research/ai-auto-tagging-solutions.md)

## 背景

素材库需要素材上传后的自动打标与语义搜索能力(产品重心「标签盲搜」的延伸)。约束:LAN 生产可完全离线(ops-manual.md:602)、医美素材隐私不出内网、单卡 3080(10GB)起步、底座级健壮性。

## 候选

1. **云端多模态 API** — 零运维,但违反离线与隐私约束,否决。
2. **模型嵌进 ms-worker**(进程内 torch) — 部署最省,但 ML 崩溃/显存泄漏直接拖垮缩略图等关键任务,违反健壮性要求,否决。
3. **独立 ML 微服务 + 队列驱动**(选定) — Immich 已验证的形态:ML 独立容器(GPU),worker 经 HTTP 调用,arq 队列解耦,故障域隔离。
4. 向量库:Qdrant/Milvus(重,百万级才需要)vs **pgvector**(复用现有 PG16,零新增组件)。

## 决策

- 架构:独立 ms-ml 服务(vLLM 跑 Qwen2.5-VL 量化版打标 + 轻量 embedding 模型),ms-ai-worker 独立 arq 队列消费,HTTP 解耦。
- 数据:AI 标签独立成区(新表,带 model_version),不直接写 user_labels;语义向量存 pgvector,与既有 pg_trgm 关键词做 RRF 混合检索。
- 扩展:模型配置带 device 字段,多卡/新模型/新媒体类型只改配置与新增任务 kind。

## 影响

- 新增:ms-ml / ms-ai-worker 两个容器、4 张表、pgvector 扩展、模型权重离线导入机制(项目首例)。
- 主系统零侵入:上传/现有搜索在 ML 全挂时行为不变;权限模型不动(搜索沿用 list_objects 过滤)。
- 回退成本:摘掉进队钩子 + 下线两容器即恢复原状,数据可整表弃置。
