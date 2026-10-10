# MiniMax H3 低显存本地部署笔记

**核验日期：2026-10-10** ｜ 适用对象：想在 6–8GB 显存的消费级显卡上本地跑 MiniMax H3 的人

> 本仓库为个人调研笔记（非官方资料）。状态类信息会过期，使用前请复核。

一句话结论：**6GB 显存只有一条路（WanGP，480p）；8GB 是社区工具的事实底线（FreeVideo）；官方部署是 4 卡级。** 我们在 6GB 笔记本上完成调研后决定「不装、存档」——这份笔记就是存档本身。

## TL;DR

| 显存档位 | 可用路线 | 能出的档位 | 速度参考 |
|---|---|---|---|
| 4×GPU（官方） | SGLang（`--num-gpus 4`） | 768p / 2K | 数据中心级 |
| 11GB+ | WanGP | 1080p | 官方 README：1080p 15s 需 11GB |
| 8GB | WanGP / FreeVideo | 544p–768p | 5–10s 视频约 8–21 分钟/条（社区实测表） |
| 6GB | 仅 WanGP | 480p | 5s 约 15–30 分钟/条（估算，未实跑） |

速度强相关于显存带宽与散热；笔记本普遍降频。

## 1. H3 是什么（先搞清楚在跑什么）

- **MiniMax H3 = 33B 稠密全模态「视频＋音频」联合生成模型**，不是语言模型（别和 M 系列混）。
- 单流 Omni-Transformer：约 13B 参数在 AdaLN 分支，推理时可预计算缓存、**不加载** ⇒ 社区剪枝版按 **20B** 算。
- 文本编码器用 **Qwen3-VL-32B 全量权重**（取第 50 层隐状态）——「小模型」的直觉在这里不成立，光编码器就几十 GB。
- 视频 VAE＝f16t4d24（空间压缩 16×、时间 4×、24 通道）；音频 VAE＝32kHz 立体声、40Hz 潜变量。
- 输出：短边默认 768p（2K 需闭源 H3-Regenerate-2K）、4–15 秒、24fps、11 种对话语言。
- **尚未开源**：H3-Context-IR（提示词预处理）、H3-Regenerate-2K、稀疏注意力实现（首发仅 full attention）。

## 2. 权重体积（2026-10-10 对 HF 文件清单实测统计）

| 组件 | 体积 |
|---|---|
| DiT 33B（官方全量 bf16） | 66.28 GB |
| DiT 剪枝 20B bf16 | 40.23 GB |
| 剪枝 20B int8-fp8 | ~21 GB |
| 剪枝 20B w6a8 | 15.98 GB |
| 剪枝 20B GGUF Q4_K | 11.42 GB |
| 剪枝 20B GGUF Q2_K | 6.72 GB |
| 文本编码器 bf16 全量 | 51.51–66.8 GB |
| 文本编码器 int8 / nvfp4 / GGUF Q4 / GGUF Q2 | 27.14 / 15.69 / 18.22 / 13.10 GB |
| 视频 VAE | 2.81–10.42 GB |
| 音频 VAE | 0.61 GB |
| 4 步 Turbo LoRA | 1.96 GB |

官方 HF 仓库文件总量约 **498.5 GB**；单任务族 bf16 完整推理也要 **≈140GB+**。低显存路线靠「剪枝 20B ＋ 低位量化 ＋ 分块/流式加载」把常驻显存压到几 GB。

## 3. 三条路线对照（引用均为原文）

| 路线 | 最低显存（原文） | 说明 |
|---|---|---|
| 官方（SGLang） | `--num-gpus 4` | 单机多卡；非消费级场景 |
| FreeVideo（FlashML-org，ComfyUI 插件，VDN-H3 8 步蒸馏） | "runs MiniMax H3 in as little as **8 GB** of VRAM and 16 GB of RAM" | 其执行规划器实测网格＝8–32GB 显存；6GB 不在支持面 |
| WanGP（Wan2GP / deepbeepmeep） | "only **5-6GB** of VRAM to gen 15 seconds of H3 video at **480p**"（v17.17） | 覆盖 6GB 的唯一路线；1080p 需 11GB |

补充：`Comfy-Org/MiniMax-H3`、`unsloth` GGUF 等提供重打包/量化权重；4–8 步蒸馏（Turbo LoRA / VDN）是低显存可用的关键。

## 4. 6GB 档的现实（我们的核验机器）

- 机器：RTX 4050 Laptop **6GB**（6141MiB）/ 31.3GB RAM / Win11 / 数据盘 224GB 空闲（实测 2026-10-10）。
- 可行组合：WanGP ＋ 20B 剪枝量化（Q4_K 11.4GB 级或更低）＋ 低位编码器，下载集约 **35–45GB**。
- 预期速度：按 8GB 档社区实测向 480p 外推，**5 秒片段约 15–30 分钟/条**（估算，未实跑）。
- 同类机器实测锚点（FreeVideo 社区表，2026-10-10 抓取）：

| 机器 | 档位 | 实测耗时 |
|---|---|---|
| 4070 Laptop 8GB / 32GB RAM | 768p 8s「Light」 | ≈830s |
| 4070 Laptop 8GB | 544p 5s | ≈590s |
| 4060 Laptop 8GB / 64GB RAM | 768p 10s | ≈1250s |
| 3070 Laptop | — | 3821s（热降频） |
| 5050 Laptop / 仅 16GB RAM | 544p 3s | ≈2130s（硬盘 80MB/s 读盘瓶颈） |

- **结论**：6GB 能跑，但只到 480p、且是「排产级」速度。我们最终决定本机不装（性价比低），把这份调研存档备用。

## 5. 许可要点（2026-10-10 对 LICENSE 原文核对）

MiniMax H3 Community License —— **非 OSI 开源**：

- 适用地域＝全球**除**欧盟、英国、韩国、美国（中国大陆在范围内）。
- 商业年收入 > 2000 万美元需向 MiniMax 取得**书面授权**（api@minimax.io）。
- 商用产品/服务的界面须**显著标注 “MiniMax H3”**。
- 附可接受使用条款（Acceptable Use），地域限制同样约束**输出物**。

（以上为要点摘录，不构成法律意见；以 LICENSE 原文为准。）

## 6. 下载清单（换到 11GB+ 显存再跑也不迟）

- 国内镜像：Hugging Face 走 `hf-mirror.com`（设 `HF_ENDPOINT=https://hf-mirror.com`），本机实测可用。
- 参考起步组合（8GB+ 机器）：20B 剪枝 fp8/int8（~21GB）＋ 编码器 nvfp4/int8（15.7–27GB）＋ 视频 VAE int8（2.8GB）＋ 音频 VAE（0.61GB）＋ 4 步 Turbo LoRA（1.96GB）。
- 主要来源仓库：
  - [MiniMaxAI/MiniMax-H3](https://huggingface.co/MiniMaxAI/MiniMax-H3)（官方权重）
  - [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3)（ComfyUI 重打包＋低位量化）
  - [deepbeepmeep/Wan2GP](https://github.com/deepbeepmeep/Wan2GP)（WanGP）
  - [FlashML-org/FreeVideo](https://github.com/FlashML-org/FreeVideo)
  - [OpenVDN/vdn-minimax-h3](https://huggingface.co/OpenVDN/vdn-minimax-h3)（VDN 蒸馏）

## 7. 避坑提醒

- 对「**新注册小号 ＋ 0 星 ＋ README 抄官方 ＋ 只发安装包下载**」形态的仓库保持默认警惕。核验期间遇到一个 0★、创建于 2026-10-06 的 `MiniMax-H3-8gb` 仓库：README 是官方文档复制、只提供安装包（.7z/.run/.dmg）下载——我们不采用，也不背书。
- 「5–10 分钟出片」一类宣传，请对照社区实测表看具体机器档位。
- 下载只认官方与知名社区源（见 §6 列表），第三方「一键包」自行承担供应链风险。

## 验证方法（可复现）

- 硬件：`nvidia-smi --query-gpu=name,memory.total --format=csv,noheader`
- 权重体积：对 `hf-mirror.com/api/models/<repo>?blobs=true` 的文件清单逐项求和
- 速度锚点：FreeVideo 仓库 `docs/community-results.md` 表格原文
- 许可：MiniMaxAI/MiniMax-H3 仓库 LICENSE 原文

## 数据来源

- MiniMaxAI/MiniMax-H3（GitHub + Hugging Face）：规格、架构、LICENSE
- deepbeepmeep/Wan2GP：README（v17.17）
- FlashML-org/FreeVideo：README ＋ docs/community-results
- 核验时间：2026-10-10

## License

本笔记文字内容以 [MIT](LICENSE) 发布；引用的第三方 README/LICENSE 片段版权归原作者。模型权重许可以 MiniMax H3 Community License 为准。
