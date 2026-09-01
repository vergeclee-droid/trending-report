---
title: "Spark X2.5-1.7B — 實測 Benchmark (penguin 10.5.255.194)"
type: benchmark
product: Edge-AI-Home-Products
tags: [verge, spark, xhtoken, benchmark, llama.cpp, edge-ai, 1M-context]
created: 2026-09-02
status: complete
---

# Spark X2.5-1.7B 實測 Benchmark

> 測試日期: 2026-09-02 | 測試機: penguin (10.5.255.194, chrislee-VMware20-1)
> 硬件: 16 cores / 23GB RAM | 軟件: XHToken llama.cpp fork (spark2_5 arch support)

## 測試設定

| Item | Value |
|------|-------|
| 模型 | Spark-X2.5-1.7B (官方 BF16 GGUF, 3.2GB) |
| GGUF | v3, 226 tensors, 37 KV pairs |
| Server | llama-server @ :8084, 4 slots × 32K ctx |
| Threads | 12 |
| 來源 | ModelScope (官方 mirror, 比 HF 快 2x) |
| Fork | github.com/XHToken/llama.cpp (必須 — 標準 llama.cpp 唔支援 spark2_5 arch) |

## Benchmark 結果

### 生成速度 (BF16, CPU)
| 測試 | Prompt tok/s | Generation tok/s | 備註 |
|------|:---:|:---:|------|
| 短問答 (150 tok) | 55.6 | **7.53** | 有 reasoning mode |
| 中文短文 (88 tok) | 46.3 | **5.63** | reasoning off, 正式輸出 |
| 長生成 (400 tok) | 51.4 | 6.75 | reasoning on 會食晒 budget |

### 能力驗證
| 測試 | 結果 |
|------|------|
| 1M context 聲稱 | ✅ config.json `max_position_embeddings: 1048576` (官方確認) |
| 繁體中文輸出 | ✅ 流暢、語義正確 (智慧家居短文) |
| 多輪對話記憶 | ✅ 記得「隻貓叫咪咪」 |
| Reasoning mode | ✅ 自動 thinking, 可 `--reasoning off` 關 |
| 架構 | Spark2_5ForCausalLM, 28 layers, 2048 hidden, 131072 vocab |

## Edge AI 評估

### RK3588 Hub 適用性
- BF16 3.2GB — 8GB RK3588 勉強 (加 KV cache 會爆), **建議 Q4_K_M (~1GB)**
- CPU-only 7.5 tok/s — 互動可接受, 但 Q4 會快 ~2x
- **1M native context 係質變** — 但 24GB RAM 機都要 32K ctx 先跑得郁, RK3588 實際用 8K-16K
- 全場景: 家庭長期記憶、整本書載入、長對話 — 唔使外部 RAG

### 對比現有方案
| 維度 | Spark 1.7B BF16 | gemma-4-E2B Q4 (現用) | Qwen3-1.7B |
|------|:---:|:---:|:---:|
| Context | **1M native** | 128K | 32-40K |
| 語言 | 中/英 + 200 語言 | 中/英 | 多語 |
| Reasoning | ✅ 內建 | 部分 | 部分 |
| License | Apache-2.0 | ? | Apache-2.0 |
| Agent 優化 | ✅ 官方主打 | - | - |

## 部署建議

1. **短期**: 用 Q4_K_M 版本 (下載中), RK3588 上試 8K-16K ctx — 速度會 ~15 tok/s
2. **中期**: 官方 MaaS API 限時免費 — 先做場景 benchmark 再決定本地化
3. **長期**: Hub 量產候選 — 1M context + reasoning 對 Home Hub / Kitchen Agent 有優勢
4. **後摩 M50**: Spark 官方支持後摩硬件 — 同 Houmo-M50-Cross-Reference 路徑吻合

## 部署紀錄 (penguin 10.5.255.194)

```bash
# 模型
~/models/spark-x2.5/Spark-X2.5-1.7B-BF16.gguf  (3.2GB, ModelScope)
~/models/spark-x2.5/Spark-X2.5-1.7B-Q4_K_M.gguf (下載中)

# fork (必須)
~/llama.cpp-spark/  (XHToken fork, spark2_5 arch)

# server
llama-server -m ~/models/spark-x2.5/Spark-X2.5-1.7B-BF16.gguf \
  --host 0.0.0.0 --port 8084 --ctx-size 32768 --batch-size 512 -t 12 \
  [--reasoning off]   # 關 thinking mode 攞直接輸出

# 測試 API
curl http://127.0.0.1:8084/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"..."}]}'
```

## Sources
- 視頻號 @算力炼丹炉 2026-09-01: https://weixin.qq.com/sph/ACgRJxhYex
- GitHub: https://github.com/XHToken/Spark-X2.5
- ModelScope: https://modelscope.cn/models/XHToken/Spark-X2.5-1.7B-GGUF
- 官方 README: 1M context (1,048,576 tokens), 200+ 語言, vLLM 單 GPU
