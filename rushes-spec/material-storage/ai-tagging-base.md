# AI 打标底座:素材自动识别与可搜索属性

- 状态: proposed
- 日期: 2026-09-15
- Issue: 待回填
- 关联: [ADR-0009](./decisions/0009-ai-auto-tagging-base.md)、调研依据 [research/ai-auto-tagging-solutions.md](./research/ai-auto-tagging-solutions.md)

## 1. 背景与目标

素材库的产品重心之一是「标签盲搜」(ROADMAP / CLAUDE.md),当前搜索仅覆盖 filename / user_labels / notes 三路文本(`api/app/routers/assets.py:398-490`),标签全靠人工。服务器有一张显卡(3080,按 10GB 显存规划;设计留多卡扩展),目标:

1. 素材上传后**自动**被模型识别,产出标签与结构化属性(场景/颜色/图中文字/描述);
2. 这些产出**可被搜索**:关键词命中 + 自然语言语义搜索(以文搜图);
3. 这是一套**底座**:健壮性第一,媒体类型与模型可扩展,ML 故障不拖累主系统。

已确认的 scope(2026-09-15 与用户确认):语义搜索要做 / AI 标签独立成区(不直接灌 user_labels)/ 一期只做图片 / 单卡 3080 起步、留多卡引子。

## 2. 现状(代码锚点)

- 上传:MinIO multipart presign 直传,complete 时 `head_object` 落库 INSERT assets(`assets.py:52/93/110/149-162`)。
- assets 表(`api/app/db/tables.py:129-177`):`media_metadata` JSONB(:143,**全库无写入方,空预留**)、`tags` JSONB(:144,缩略图派生数据)、`user_labels` ARRAY(String(64))(:146)。
- 上传后钩子:complete 按类型 enqueue 缩略图任务(`assets.py:207-218`)——**AI 打标入队的现成挂载点**。
- 队列:arq + Redis,pool 挂 `app.state.arq_pool`(`api/app/main.py:52`、`api/app/services/arq_pool.py:19-52`,enqueue fire-and-forget 失败仅 log);worker 4 个任务全在 `api/app/workers/main.py`,`max_jobs=4, job_timeout=60s`(:650-661),arq 支持按函数配置 timeout / queue。
- 失败处理模式:任务内 catch,失败写 `tags['thumbnail_failed']` 只存异常类型名(`workers/main.py:148-157`);幂等靠产物 key 固定覆盖写。
- 回填先例:`api/scripts/backfill_thumbnails.py`(扫全表 enqueue,--force 重跑)。
- 搜索:`GET /assets/search` 权限先行(OpenFGA `list_objects(can_view)` 圈 folder 集合,`assets.py:436-457`)+ 三路 ILIKE 走 pg_trgm GIN(`assets.py:461-475`、`tables.py:164-176`);sensitive 素材存在性零泄露是硬验收。
- 部署:LAN 单机可完全离线(ops-manual.md:602/654);compose 有 `APT_MIRROR`/`PIP_INDEX_URL` 构建参数(docker-compose.yml:42-45);**全库零 AI 调用痕迹、零 GPU 配置、无模型权重离线分发先例**(2026-09-15 全仓 grep 实证)。

## 3. 总体架构

```
上传 complete ──enqueue──▶ Redis(arq,queue=ai)
                               │
                    ms-ai-worker(新容器,无 GPU,同 api 镜像)
                               │ HTTP(httpx,已有依赖)
                    ms-ml(新容器,GPU,vLLM + ONNX)
                               │
                    回写 PG:asset_ai_labels / asset_ai_attrs / asset_embeddings / ai_jobs
```

- **ML 独立成服务**:进程隔离、显存隔离、崩溃域隔离;ms-ai-worker 不装 torch,只经 HTTP 调 ms-ml(参考 Immich 架构形态)。
- **AI 标签独立成区**:不碰 `user_labels`;前端分开展示,可一键「采纳」写入 user_labels(走现有 `PATCH /assets/{id}/meta`,assets.py:494-557)。
- **结果全部带 model_version**:换模型 = 新版本行,旧行保留,可全量重打、可区分来源;搜索只命中 settings 里的 active 版本。

## 4. 数据模型(新 migration)

| 表 | 关键列 | 约束/索引 |
| --- | --- | --- |
| `ai_jobs` | asset_id FK、kind(tag/embed)、status(pending/running/done/failed)、attempts、last_error(仅异常类型名)、model_version、timestamps | (asset_id,kind,model_version) 唯一;status 索引(回填/重试扫描用) |
| `asset_ai_labels` | asset_id FK CASCADE、label varchar(64)、confidence real、model_version | PK(asset_id,label,model_version);label 上 pg_trgm GIN |
| `asset_ai_attrs` | asset_id、attrs JSONB(description/scene/colors/ocr_text)、model_version | PK(asset_id,model_version) |
| `asset_embeddings` | asset_id、embedding vector(N)、model_version | PK(asset_id,model_version);HNSW 索引;`CREATE EXTENSION vector` 进 ms-db |

## 5. ML 服务与模型选型(3080 10GB 预算)

ms-ml 容器(FastAPI 薄封装,`runtime: nvidia`):

| 能力 | 一期选型 | 显存估算 |
| --- | --- | --- |
| 打标+属性(VLM) | Qwen2.5-VL-7B-Instruct AWQ(4bit),vLLM serve,guided JSON 输出 | ~6GB |
| 语义向量 | SigLIP2 multilingual 或 Qwen3-VL-Embedding-2B(P0 实测定档) | 1–2GB |
| 兜底快打标(可选) | RAM++ ONNX | ~0.5GB,可 CPU |

合计 ~8–9GB,紧张但可行;不足时降级路径:VLM 换 3B / embedding 换 SO400M / 按任务懒加载卸载。**P0 必须在 3080 上实测显存与吞吐并回填本文档。**

- 打标输入:直接用现有 1024px 缩略图(`thumbnails/{asset_id}.jpg`),无缩略图时拉原图降到 2048px——不送原片,省带宽省显存。
- VLM 输出契约:guided JSON `{tags:[{label,confidence}], description, scene, colors[], ocr_text}`;经版本化 label_map(英文→中文别名、白/黑名单)落库。
- **多卡引子**:settings 里每模型一条 `{name, device: "cuda:N"}` 配置,ms-ai-worker 按 device 路由任务;compose 预留多 device 映射。加卡 = 改配置 + 多起 worker,不改代码。

## 6. 任务队列与健壮性(本 spec 的重心)

1. **上传零阻塞**:enqueue 沿用现有 fire-and-forget(arq_pool.py:31-52),ML 全挂时上传主流程无任何感知。
2. **独立队列与并发**:AI 任务走独立 arq queue(`_queue_name="ai"`),由新容器 ms-ai-worker 独占消费(同 api 镜像,`max_jobs=1`,单卡串行;多卡后按 device 多起);AI 函数 per-function `timeout=600s`(现全局 60s 不够用)。
3. **健康闸**:任务开工前探 ms-ml `/healthz`,不健康 → arq Retry 延迟重试,**不计失败**;健康恢复后 backlog 自动消化。
4. **重试与终态**:arq max_tries=3 + backoff;最终失败落 `ai_jobs.failed`,附异常类型名(沿用 thumbnail_failed 的信息口径,不记路径/内容),可 admin 重试。
5. **幂等**:回写一律按 PK upsert;同 asset 同 model_version 重跑行数不增(硬验收)。
6. **回填/重打**:`scripts/backfill_ai.py` 仿 `backfill_thumbnails.py`,`--model-version` 指定新版本即全量重打,旧版本行保留。
7. **观测**:ai_jobs 表即事实源(pending 堆积 = 队列深度,failed 率),healthz 汇总;后续接 metrics 不堵路。
8. **大文件/异常输入**:缩略图缺失、图片解码失败、VLM 输出非法 JSON 都落明确 failed 原因,不 crash worker。

## 7. 搜索接入

- **关键词分支**:`/assets/search` 三路 ILIKE 加第四路——`EXISTS (asset_ai_labels WHERE label ILIKE … AND model_version = active)`,沿用同一 folder 权限过滤(assets.py:436-457),sensitive 零泄露不变。
- **语义分支**:query → ms-ml `/embed` → `asset_embeddings` HNSW cosine topK(active 版本)→ 与关键词结果 **RRF 融合**;权限过滤先行,语义结果在 UI 标注「智能匹配」。
- AI 属性(description/ocr_text 等)纳入关键词匹配的候选字段,权重低于标签。

## 8. 离线分发(新机制,项目首个先例)

- 模型权重:有网机 `scripts/ml_models_pull.sh` 拉取钉版权重(sha256 校验)→ tar → 内网机导入挂卷 `./ml-models`;文档进 ops-manual。
- 依赖:pip 沿用现有 mirror build args;完全无 egress 时镜像整体 `docker save/load`;compose 镜像钉 digest(沿用 poc/minio/docker-compose.yml:79 的做法)。

## 9. 分期实施

| 期 | 内容 | 出口 |
| --- | --- | --- |
| P0 底座管线 | 4 张表 migration、ms-ml 骨架(stub 模型回固定结果)、ms-ai-worker、健康闸、幂等回写、backfill 脚本、权重导入机制;3080 实测显存/吞吐 | 全链路跑通(假 AI),故障注入演练通过 |
| P1 真实打标 | VLM 接入 + label_map、素材详情 AI 标签区展示、盲搜关键词第四路 | 传图出真标签,可搜 |
| P2 语义搜索 | embedding 模型定档、pgvector 索引、RRF 融合、UI 标注 | 自然语言可搜图 |
| P3 治理 | 采纳→user_labels、驳回、受控词表、批量重打 UI | 标签质量闭环 |

视频抽帧打标 / Whisper 转写 / 人脸聚类:不在本 spec,底座已留接口(ai_jobs.kind 可扩、media 分派点即缩略图分派块),需要时另立 spec。

## 10. 验收标准

1. 上传 jpg → 60s 内 AI 标签落库且搜索可命中;**停掉 ms-ml 后上传/搜索(关键词)照常**,恢复后 backlog 自动补打。
2. 同 asset 同 model_version 重跑 2 次 → asset_ai_labels 行数不增。
3. 注入 ms-ml 500 → ai_jobs 落 failed 且可重试成功;其他类型任务不受影响;last_error 只含异常类型名。
4. 语义搜索自然语言 query 返回相关素材;非授权 folder(含 sensitive)结果零出现、存在性零泄露(沿用盲搜硬验收口径)。
5. `backfill_ai.py --model-version 新版 --force` 后:旧版本行保留、搜索只命中 active 版本。
6. 无 egress 内网机按 ops-manual 新章节从零部署 → 全部 healthz 绿。

## 11. 非目标与风险

非目标:训练/微调、云端 API、视频与人脸(见 §9)、自动删除/审核素材。

风险与对策:VLM 幻觉 → guided JSON + 置信度 + 独立标签区隔离 + P3 采纳工作流;10GB 显存紧张 → §5 降级路径,P0 实测定档;中文 embedding 质量参差 → P0/P2 候选对比实测;pgvector 规模上限 → 百万级再迁专用库,迁移成本可控(向量可全量重灌);隐私 → 全部本地推理,素材不出内网。

## 12. 开放问题

- 3080 是否独占(同机是否有其他 GPU 负载挤占显存)?
- 中文标签词表来源:自由生成 + label_map 归一,还是 P3 直接上受控词表?
- embedding 模型最终选型(P0 实测后回填 §5)。
