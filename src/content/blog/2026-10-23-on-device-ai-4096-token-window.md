---
title: 4096 个 token 的上下文窗口，工具定义也占预算
subtitle: 端侧模型不是云端的缩水版，而是请求进入系统后的一道本地闸门；它省下什么、又把什么留给手机，答案藏在窗口、接口和设备名单里。
date: '2026-10-23'
category: Deep Dive
type: deep_dive
tags:
- 端侧AI
- Apple Intelligence
- Gemini Nano
- 上下文窗口
- 移动推理
summary: Apple 的 Foundation Models 文档把默认端侧模型的单会话上下文限定为 4096 tokens，提示、工具定义、工具返回和模型回答都从这笔预算里扣。围绕这个窗口，Apple
  与 Google 把端侧模型做成系统 API，用窄任务接口、设备门槛和模型压缩控制可用范围。独立研究则提醒，本地推理仍要消耗电量、内存和热预算，省掉云端调用不等于省掉系统总成本。
tldr: ''
duration: 807.024
audioUrl: /audio/2026-10-23-4096-%E4%B8%AA-token-%E7%9A%84%E4%B8%8A%E4%B8%8B%E6%96%87%E7%AA%97%E5%8F%A3-%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%E4%B9%9F%E5%8D%A0%E9%A2%84%E7%AE%97.mp3
audioPrivate: false
visibility: public
sources:
- title: Introducing Apple’s On-Device and Server Foundation Models
  url: https://machinelearning.apple.com/research/introducing-apple-foundation-models
  publisher: Apple Machine Learning Research
  publishedAt: '2024-07-29T00:00:00Z'
- title: Apple Intelligence Foundation Language Models Tech Report 2025
  url: https://machinelearning.apple.com/research/apple-foundation-models-tech-report-2025
  publisher: Apple Machine Learning Research
  publishedAt: '2025-07-17T00:00:00Z'
- title: Updates to Apple’s On-Device and Server Foundation Language Models
  url: https://machinelearning.apple.com/research/apple-foundation-models-2025-updates
  publisher: Apple Machine Learning Research
- title: WWDC25 macOS guide - Discover
  url: https://developer.apple.com/news?id=uf3qdaoe
  publisher: Apple Developer
  publishedAt: '2025-06-09T00:00:00Z'
- title: Foundation Models
  url: https://developer.apple.com/tutorials/data/documentation/foundationmodels.json
  publisher: Apple Developer Documentation
- title: Managing the context window
  url: https://developer.apple.com/tutorials/data/documentation/foundationmodels/managing-the-context-window.json
  publisher: Apple Developer Documentation
- title: 'Google I/O 2024 recap: Making AI accessible and helpful for every developer'
  url: https://developers.googleblog.com/en/google-io-2024-recap-making-ai-accessible-and-helpful-for-every-developer/
  publisher: Google Developers Blog
- title: Send custom requests to Gemini Nano with GenAI Prompt API
  url: https://developers.google.com/ml-kit/genai/prompt/android
  publisher: Google for Developers / ML Kit
- title: Do More With Google on Android Phones & Devices
  url: https://www.android.com/
  publisher: Android / Google
- title: 'Introducing Gemma 3n: The developer guide'
  url: https://developers.googleblog.com/en/introducing-gemma-3n-developer-guide/
  publisher: Google Developers Blog
  publishedAt: '2025-06-26T00:00:00Z'
- title: MLCommons Releases MLPerf Mobile v6.0 with New Generative AI Benchmarks for
    On-Device LLMs
  url: https://mlcommons.org/2026/06/mlperf-mobile-v6
  publisher: MLCommons
  publishedAt: '2026-06-15T00:00:00Z'
- title: Dissecting the Impact of Mobile DVFS Governors on LLM Inference Performance
    and Energy Efficiency
  url: https://arxiv.org/html/2507.02135v1
  publisher: arXiv；上海交通大学与 Purdue University 作者团队
  publishedAt: '2025-07-02T20:47:40Z'
- title: 'LLM Inference at the Edge: Mobile, NPU, and GPU Performance Efficiency Trade-offs
    Under Sustained Load'
  url: https://arxiv.org/html/2603.23640v2
  publisher: arXiv；Conscious Engines 作者团队
  publishedAt: '2026-06-07T18:38:40Z'
- title: 'LLM Inference at the Edge: Mobile, NPU, and GPU Performance Efficiency Trade-offs
    Under Sustained Load'
  url: https://arxiv.org/abs/2603.23640
  publisher: arXiv；Conscious Engines 作者团队
  publishedAt: '2026-06-07T18:38:40Z'
chapters: []
keyPoints: []
---

## 4096 个 token 的窗口，先装什么

打开 Apple 的 Foundation Models 开发者文档，“Managing the context window”一页写得很干脆：默认端侧模型单个 session 的上下文窗口是 4096 tokens。这是截至 2026 年 10 月 23 日的文档口径。它不是参数规模，也不是可用内存，而是这个会话能同时看见的全部文本预算：系统指令、用户提示、工具定义和参数结构、工具调用的入参与返回、结构化输出的格式声明，以及模型自己写出来的回答，都从这 4096 里扣。

把这句话翻成一个具体请求。假设你在一个记事类 App 里让手机模型“读完这段会议纪要，标出待办、负责人和截止日期”。开发者为它挂了三个工具：一个写日历、一个查联系人、一个存备忘。做个假设算例，不是实测：每个工具的定义和参数结构按 300 token 估，三个合计 900；会议纪要若是一千五百字的中文，Apple 文档提示中日韩文字常接近一字一 token，那又是一千五；再留 400 token 给结构化输出，三项相加约 2800，窗口里还剩约 1296。系统指令、历史消息和上一轮工具返回还没算进去，所以这个算例看起来宽裕，真实占用仍要按实际文本和工具定义逐项计数。工具定义确实占用预算，但具体分走多少，没有样本就不能下结论。

更关键的是预算耗尽时的行为。Apple 文档说，达到 context size 后，session 不能继续处理更多请求，建议开发者裁剪历史或创建新的 session。开发者若没有处理这个限制，用户可能看到一次失败或空白结果；这是应用层的错误处理问题，不是模型在暗地里改写任务约定。工具越多、工具定义和参数结构 越复杂，越容易提前撞到这条线，于是裁剪历史、重建 session、把长任务拆段，成了端侧开发的日常动作。

Apple 的框架文档另处建议：需要更强推理或更大上下文时，使用 Private Cloud Compute 或其他服务器模型（C07，Apple Developer 文档，2026-10-23 口径）。这是一条开发者可以采用的方案，不是系统自动把装不下的请求往上传。文档没有公布路由阈值、回退比例或实际请求占比，所以更准确的说法是：应用可以自己决定哪些任务留在本地、哪些交给服务器。端侧模型在这里的角色，是应用选择留在端侧的请求的本地执行层，而不是云端模型的全面替代品。

本文涉及的参数量、内存、token 和吞吐分别来自厂商披露、开发者文档与独立预印本，彼此不能直接换算或横比；厂商自报数字与独立评测也分开处理。这个口径只在第一次出现时集中交代，后面不再反复声明。

## 两把减法刀：权重压窄，缓存复用

Apple 2024 年披露的架构是两条腿：约 30 亿参数的端侧语言模型，加一个跑在 Private Cloud Compute 上的更大服务器模型（C01，2024-07-29 更新的介绍页）。2025 年的技术报告进一步说，这个 3B 端侧模型针对 Apple silicon 优化，用了 KV-cache sharing 和 2-bit 量化感知训练（C02，2025-07-17 技术报告）。

2-bit 量化感知训练值得拆开。生成阶段常常受内存带宽限制：每吐一个 token，都要把大量权重从内存搬进计算单元。在内存带宽受限的解码路径上，权重更窄可能减少每个 token 的搬运字节；实际速度还取决于硬件、kernel 和批处理条件。量化感知训练的做法是在训练时就模拟低精度误差，让模型学会在 2-bit 下仍然可用，而不是训练完再粗暴地四舍五入。省下的是权重占用的存储和带宽，代价可能是精度余量被提前花掉一部分；但 2-bit 是否必然带来可感知的质量下降，要看具体模型和任务，不能从位宽直接推断。

KV-cache 更值得细看。模型回答时，每一层都要缓存已处理 token 的 Key 和 Value，避免每次都重算整段历史。序列越长、层数越多，这块缓存越大，而且它随上下文线性膨胀。在长上下文场景，KV-cache 可能成为内存预算里增长最快的一项。Apple 报告称，通过让不同模型块共享 KV cache，这块缓存的内存用量降了 37.5%（C03，2025 年模型更新）。口径要卡死：这是 KV-cache 这一块的架构内存，不是整机 RAM 降 37.5%，也不是每次推理耗电降 37.5%。

两把刀合起来，是端侧模型的选择性瘦身：权重压窄、缓存复用，把最吃内存的生成阶段塞进手机预算里。代价同样清晰——一个约 30 亿参数、采用 2-bit 量化感知训练的模型，本来就不是拿来跟云端大模型比推理深度的。它被设计成一个在本地预算内可运行的前置层；是否够快、够省，要看具体任务和机型，不能从参数或位宽直接推断。

## 模型从产品功能变成系统服务

2024 年 Google I/O，Google 把 AICore 描述成管理端侧基础模型的系统服务，主打的是“应用不用自己管大模型分发”，当时披露 Gemini Nano 已在 Pixel 8 Pro 与 Galaxy S24 系列提供（C08，2024 年 I/O 回顾）。这一步改的是分发层：模型的生命周期从每个 App 自己打包，挪到系统统一管理。

旧方案是应用各自携带模型或各自对接推理栈，集成门槛高，版本也难统一。AICore 把这件事收进系统服务，应用只调用接口，受益的是不想维护基础模型的开发者。代价是可用性被系统版本和硬件名单绑住：同一段代码，在不同机型上可能根本找不到可用的本地模型。这为后面的设备门槛埋下伏笔。

2025 年 6 月 9 日 WWDC25，Apple 则把系统级模型能力开放成开发者接口。Foundation Models framework 让第三方开发者直接调用 Apple Intelligence 的核心端侧模型（C04，2025-06-09）。受益者同样是开发者：不用自己部署和微调一个基础模型，调系统接口就行。两家都提供了系统级入口，但 Google 侧重模型分发管理，Apple 侧重模型调用接口。

但接口一旦标准化，能力边界也一起写进文档。Apple 把 4096 的预算摆上台面，等于承认本地这层执行器只适合装得下的任务；需要更大上下文或更强推理时，开发者要自己选择服务器方案。端侧与云端的分工，从模型架构问题变成了产品接口问题。

平台产品接口的比较限于 Apple 与 Google 已公开的开发接口和产品文档；性能证据另引 MLCommons 与独立预印本，不覆盖 OEM、Qualcomm/MediaTek 运行时或市场份额。

## 窄接口换可控，设备名单换覆盖

Google 的 ML Kit GenAI Prompt API 是一面镜子。它接收文本或“图像加文本”，输出文本或结构化结果；但专用任务 API 收得很紧——摘要只承诺 1–3 个要点，改写只能用预设风格，图像描述只给通用短描述（C09，Google 开发者文档，2026-10-23 口径）。

这不是能力不足的补丁，而是把模型约束在可验证的输出形态里。要点数量固定、风格预设、描述短，意味着应用能预先知道返回结构，出错时也容易判断是模型问题还是数据问题。窄接口用通用性换来了输出形态的可约束性，集成边界更明确；这不保证正确性，也不等于能自动定位错误原因。需要强调的是，这些限制针对对应的 feature-specific API，不能泛化成 Gemini Nano 所有调用方式的能力上限。

门槛还有硬件那一层。Android 官网把 Gemini Intelligence 功能限定在具备 AI Core 与 Nano v3 以上、至少 12GB RAM、且搭载合格旗舰 SoC 的设备（C10，android.com，2026-10-23 口径）。注意范围：这是 Gemini Intelligence 这个产品功能的要求，不能外推成“所有 Nano 调用都要 12GB”。

两个事实合起来是一笔覆盖账。一个端侧模型能触达多少用户，取决于系统服务版本、芯片认证和内存下限。模型能不能跑是一回事，多少台在售手机被允许跑是另一回事。厂商发布会展示的是前者，用户手里的机器决定的是后者。

## 有效参数不等于原始参数

2025 年 6 月 26 日 Google 发布 Gemma 3n，命名用的是 E2B/E4B。Google 开发者博客自己把话说透：“While their raw parameter count is 5B and 8B respectively, architectural innovations allow them to run with a memory footprint comparable to traditional 2B and 4B models.”（Google Developers Blog 作者，2025-06-26，C11）。也就是说，E2B/E4B 说的是有效参数，原始参数是 5B 和 8B；Google 称架构优化让它们跑到接近传统 2B/4B 模型的内存占用，最低内存分别到 2GB/3GB。

Google 对 Gemma 3n 的公开说明只支持到有效参数、原始参数和最低内存这几个口径；E2B/E4B 是已发布的子模型，Mix-n-Match 可以组合尺寸。至于模型内部如何按输入选择路径，公开材料没有给出足够细节，本文不展开。运行时真正参与计算的量可能变小，权重总量却不一定小，所以有效参数既不是权重总量，也不是设备所需内存的同义词，三个口径不能互相换算。2GB/3GB 也是 Google 的部署主张，不是独立手机实测的统一结果。

这一步的意义在于，竞争开始从“把模型做小”转向“让部署按档位可调”。同一套工具链里，开发者可以按设备和任务选择不同尺寸，而不是只拿到一个固定模型。但它也带来新的混淆：宣传页上的数字到底指哪一层，开发者要自己问清楚。

## 免费的是调用，不是电

Apple 的 WWDC25 开发者指南给了一句传播很广的话：“And it's all available at no cost for any request.”（Apple Developer editorial team，2025-06-09，C05）。这句话成立的范围很窄：用户和开发者不按请求付平台调用费，而且请求可以离线跑。它没说电量、热预算、内存占用和设备折旧为零，也没说工作流不会另行调用云端。

独立研究正好补上这块。一篇 2025 年 7 月 2 日的移动 DVFS 预印本（arXiv 2507.02135 v1，上海交大与 Purdue 作者团队）在 Pixel 7 与 Pixel 7 Pro 等测试平台上，抽样了若干预填充和解码长度。论文报告：在相同能耗条件下，手机默认的 CPU、GPU、内存调频组合，时延最多比它测试的较优频率组合高 40.4%；作者的 FUSE 方案在 ShareGPT 评测里，TTFT 平均改善 7.0%–16.9%，TPOT 平均改善 25.4%–36.8%，并称每 token 能耗相同（C13）。

这些数字背后的机制，按论文自己的说法，是 CPU、GPU 和内存的调频器各自独立运行，可能选到次优频率，并且彼此影响。它说明的是特定设备、特定模型和采样负载下的运行时策略问题，不是“所有手机都会在提频后迟迟不回落”的普遍规律。省下云调用费，并不自动意味着用户设备上的计算成本、发热或交互延迟更低。本地请求只是把计算搬到了手机上，电量和时间仍要付。

## 持续负载：两个页面给出两套结果

持续负载更能说明问题。一篇 2026 年 6 月 7 日的跨平台预印本（arXiv 2603.23640 v2）正文称，用 Qwen 2.5 1.5B、4-bit 量化、固定 258-token prompt，四个平台在热机条件下每台设备重复 20 轮：iPhone 16 Pro 吞吐在前三轮下降约 40% 后稳定在约 23.7 tok/s，S24 Ultra 下降约 15% 后稳定在约 10 tok/s（C14）。

但同一版本的摘要却说，iPhone 16 Pro 在两轮内吞吐下降近一半，S24 Ultra 撞上 OS 强制的 GPU 频率下限而完全终止推理（C15）。同一篇论文的两个页面互相打架。这本身也是结论的一部分：单篇预印本、单次峰值，都不足以给手机排名。在作者更正、PDF 复核或独立复现之前，只能把两组描述并列，不能挑一组当成已稳定复现的事实，也不能推测哪一页更可信。

热这件事在手机上确实硬。在本文引用的测试设备上，手机靠被动散热，持续负载可能触发降频和限功耗；不同机型的散热设计不同，不能把这当成所有手机的统一行为。DVFS 预印本讲的是调频策略在相同能耗下跑出不同时延，持续负载预印本讲的是特定平台在长时间运行中的热行为，两者涉及不同因素，不能合并成同一个失效模式。它们共同支持的，只是一个更窄的判断：本地推理的瓶颈不只是算不算得动，还包括能不能一直算下去。

## 标准开始搭台，但台子还很窄

2026 年 6 月 15 日，MLCommons 在 MLPerf Mobile v6.0 里加了 Android 端侧 LLM 测试，模型选 Llama 3.2 1B/3B Instruct 和 Llama 3.1 8B Instruct，请求从 TinyMMLU、IFEval 里取；测试可在内存足够的设备上用 CPU 跑，8B 的 NPU 加速路径在 v6.0 发布说明里列出 Snapdragon 8 Elite Gen 5（C12）。

这是把移动 LLM 从厂商自选 benchmark 拉向固定任务的一步：模型、数据集、请求都定死，成绩才有可比性。但它仍是测试套件说明，不是成绩单；它覆盖 Android、覆盖指定加速路径，不覆盖 iOS 与 Android 同条件横比，也没有给出续航和持续热负载的完整结论。标准搭起了台子，台子还很窄。

## 这台机器到底省了什么

回到最初的问题。我的判断是：对确实留在本地的请求，端侧模型可以避免相应的云推理调用、一次网络往返，以及敏感输入离开设备的概率。它没有省掉计算，只是把计算从机房的电费，换成了你手机的电量、内存和热预算。至于现有产品实际把多少请求留在本地、总共省了多少，公开数据里没有本地请求占比，也没有本地与云端的可比总成本，所以无法做可靠的总量核算。

Apple 的 session 预算、Google 特定任务 API 的输出约束、Gemini Intelligence 的设备资格，是三个不同层级的限制。它们共同提示开发者：端侧任务要按上下文、输出形态和设备名单来规划；但不能说两家产品设计完全相同。对做音频或会议记录的产品来说，这意味着端侧更适合从已转写的文本里做短摘要、分类和实体抽取，而长稿、多轮上下文和复杂推理，仍要预留分段或服务器路径。

一个更窄的判断是：在持续负载场景，“每 Wh 能稳定处理多少真实请求”可能比峰值 TOPS 或参数量更能解释体验差异。这个判断仍是推测，需要匹配质量、任务和持续条件的独立测试来验证；DVFS 论文和那篇有内部冲突的持续负载预印本，只足以提示这个方向，不足以证明行业竞争重心已经转移。

一个可核查的预测是：到 2027 年底，主流手机平台仍会用短任务接口和设备资格来承接端侧模型，长上下文与强推理继续走服务器路径。如果届时主流平台普遍在本地处理长上下文且不需要服务器回退，这个判断就该重写。在那之前，把手机模型当成云端之前的本地执行层，比当成云端的替代品，更接近可查到的证据。

<details class="transcript-fold">
<summary><span>📝 完整逐字稿</span><small>点击展开</small></summary>
<div class="transcript-body"><h3>4096 个 token 的窗口，先装什么</h3>
<p>打开 Apple 的 Foundation Models 开发者文档，“Managing the context window”一页写得很干脆：默认端侧模型单个 session 的上下文窗口是 4096 tokens。这是截至 2026 年 10 月 23 日的文档口径。它不是参数规模，也不是可用内存，而是这个会话能同时看见的全部文本预算：系统指令、用户提示、工具定义和参数结构、工具调用的入参与返回、结构化输出的格式声明，以及模型自己写出来的回答，都从这 4096 里扣。</p>
<p>把这句话翻成一个具体请求。假设你在一个记事类 App 里让手机模型“读完这段会议纪要，标出待办、负责人和截止日期”。开发者为它挂了三个工具：一个写日历、一个查联系人、一个存备忘。做个假设算例，不是实测：每个工具的定义和参数结构按 300 token 估，三个合计 900；会议纪要若是一千五百字的中文，Apple 文档提示中日韩文字常接近一字一 token，那又是一千五；再留 400 token 给结构化输出，三项相加约 2800，窗口里还剩约 1296。系统指令、历史消息和上一轮工具返回还没算进去，所以这个算例看起来宽裕，真实占用仍要按实际文本和工具定义逐项计数。工具定义确实占用预算，但具体分走多少，没有样本就不能下结论。</p>
<p>更关键的是预算耗尽时的行为。Apple 文档说，达到 context size 后，session 不能继续处理更多请求，建议开发者裁剪历史或创建新的 session。开发者若没有处理这个限制，用户可能看到一次失败或空白结果；这是应用层的错误处理问题，不是模型在暗地里改写任务约定。工具越多、工具定义和参数结构 越复杂，越容易提前撞到这条线，于是裁剪历史、重建 session、把长任务拆段，成了端侧开发的日常动作。</p>
<p>Apple 的框架文档另处建议：需要更强推理或更大上下文时，使用 Private Cloud Compute 或其他服务器模型（C07，Apple Developer 文档，2026-10-23 口径）。这是一条开发者可以采用的方案，不是系统自动把装不下的请求往上传。文档没有公布路由阈值、回退比例或实际请求占比，所以更准确的说法是：应用可以自己决定哪些任务留在本地、哪些交给服务器。端侧模型在这里的角色，是应用选择留在端侧的请求的本地执行层，而不是云端模型的全面替代品。</p>
<p>本文涉及的参数量、内存、token 和吞吐分别来自厂商披露、开发者文档与独立预印本，彼此不能直接换算或横比；厂商自报数字与独立评测也分开处理。这个口径只在第一次出现时集中交代，后面不再反复声明。</p>
<h3>两把减法刀：权重压窄，缓存复用</h3>
<p>Apple 2024 年披露的架构是两条腿：约 30 亿参数的端侧语言模型，加一个跑在 Private Cloud Compute 上的更大服务器模型（C01，2024-07-29 更新的介绍页）。2025 年的技术报告进一步说，这个 3B 端侧模型针对 Apple silicon 优化，用了 KV-cache sharing 和 2-bit 量化感知训练（C02，2025-07-17 技术报告）。</p>
<p>2-bit 量化感知训练值得拆开。生成阶段常常受内存带宽限制：每吐一个 token，都要把大量权重从内存搬进计算单元。在内存带宽受限的解码路径上，权重更窄可能减少每个 token 的搬运字节；实际速度还取决于硬件、kernel 和批处理条件。量化感知训练的做法是在训练时就模拟低精度误差，让模型学会在 2-bit 下仍然可用，而不是训练完再粗暴地四舍五入。省下的是权重占用的存储和带宽，代价可能是精度余量被提前花掉一部分；但 2-bit 是否必然带来可感知的质量下降，要看具体模型和任务，不能从位宽直接推断。</p>
<p>KV-cache 更值得细看。模型回答时，每一层都要缓存已处理 token 的 Key 和 Value，避免每次都重算整段历史。序列越长、层数越多，这块缓存越大，而且它随上下文线性膨胀。在长上下文场景，KV-cache 可能成为内存预算里增长最快的一项。Apple 报告称，通过让不同模型块共享 KV cache，这块缓存的内存用量降了 37.5%（C03，2025 年模型更新）。口径要卡死：这是 KV-cache 这一块的架构内存，不是整机 RAM 降 37.5%，也不是每次推理耗电降 37.5%。</p>
<p>两把刀合起来，是端侧模型的选择性瘦身：权重压窄、缓存复用，把最吃内存的生成阶段塞进手机预算里。代价同样清晰——一个约 30 亿参数、采用 2-bit 量化感知训练的模型，本来就不是拿来跟云端大模型比推理深度的。它被设计成一个在本地预算内可运行的前置层；是否够快、够省，要看具体任务和机型，不能从参数或位宽直接推断。</p>
<h3>模型从产品功能变成系统服务</h3>
<p>2024 年 Google I/O，Google 把 AICore 描述成管理端侧基础模型的系统服务，主打的是“应用不用自己管大模型分发”，当时披露 Gemini Nano 已在 Pixel 8 Pro 与 Galaxy S24 系列提供（C08，2024 年 I/O 回顾）。这一步改的是分发层：模型的生命周期从每个 App 自己打包，挪到系统统一管理。</p>
<p>旧方案是应用各自携带模型或各自对接推理栈，集成门槛高，版本也难统一。AICore 把这件事收进系统服务，应用只调用接口，受益的是不想维护基础模型的开发者。代价是可用性被系统版本和硬件名单绑住：同一段代码，在不同机型上可能根本找不到可用的本地模型。这为后面的设备门槛埋下伏笔。</p>
<p>2025 年 6 月 9 日 WWDC25，Apple 则把系统级模型能力开放成开发者接口。Foundation Models framework 让第三方开发者直接调用 Apple Intelligence 的核心端侧模型（C04，2025-06-09）。受益者同样是开发者：不用自己部署和微调一个基础模型，调系统接口就行。两家都提供了系统级入口，但 Google 侧重模型分发管理，Apple 侧重模型调用接口。</p>
<p>但接口一旦标准化，能力边界也一起写进文档。Apple 把 4096 的预算摆上台面，等于承认本地这层执行器只适合装得下的任务；需要更大上下文或更强推理时，开发者要自己选择服务器方案。端侧与云端的分工，从模型架构问题变成了产品接口问题。</p>
<p>平台产品接口的比较限于 Apple 与 Google 已公开的开发接口和产品文档；性能证据另引 MLCommons 与独立预印本，不覆盖 OEM、Qualcomm/MediaTek 运行时或市场份额。</p>
<h3>窄接口换可控，设备名单换覆盖</h3>
<p>Google 的 ML Kit GenAI Prompt API 是一面镜子。它接收文本或“图像加文本”，输出文本或结构化结果；但专用任务 API 收得很紧——摘要只承诺 1–3 个要点，改写只能用预设风格，图像描述只给通用短描述（C09，Google 开发者文档，2026-10-23 口径）。</p>
<p>这不是能力不足的补丁，而是把模型约束在可验证的输出形态里。要点数量固定、风格预设、描述短，意味着应用能预先知道返回结构，出错时也容易判断是模型问题还是数据问题。窄接口用通用性换来了输出形态的可约束性，集成边界更明确；这不保证正确性，也不等于能自动定位错误原因。需要强调的是，这些限制针对对应的 feature-specific API，不能泛化成 Gemini Nano 所有调用方式的能力上限。</p>
<p>门槛还有硬件那一层。Android 官网把 Gemini Intelligence 功能限定在具备 AI Core 与 Nano v3 以上、至少 12GB RAM、且搭载合格旗舰 SoC 的设备（C10，android.com，2026-10-23 口径）。注意范围：这是 Gemini Intelligence 这个产品功能的要求，不能外推成“所有 Nano 调用都要 12GB”。</p>
<p>两个事实合起来是一笔覆盖账。一个端侧模型能触达多少用户，取决于系统服务版本、芯片认证和内存下限。模型能不能跑是一回事，多少台在售手机被允许跑是另一回事。厂商发布会展示的是前者，用户手里的机器决定的是后者。</p>
<h3>有效参数不等于原始参数</h3>
<p>2025 年 6 月 26 日 Google 发布 Gemma 3n，命名用的是 E2B/E4B。Google 开发者博客自己把话说透：“While their raw parameter count is 5B and 8B respectively, architectural innovations allow them to run with a memory footprint comparable to traditional 2B and 4B models.”（Google Developers Blog 作者，2025-06-26，C11）。也就是说，E2B/E4B 说的是有效参数，原始参数是 5B 和 8B；Google 称架构优化让它们跑到接近传统 2B/4B 模型的内存占用，最低内存分别到 2GB/3GB。</p>
<p>Google 对 Gemma 3n 的公开说明只支持到有效参数、原始参数和最低内存这几个口径；E2B/E4B 是已发布的子模型，Mix-n-Match 可以组合尺寸。至于模型内部如何按输入选择路径，公开材料没有给出足够细节，本文不展开。运行时真正参与计算的量可能变小，权重总量却不一定小，所以有效参数既不是权重总量，也不是设备所需内存的同义词，三个口径不能互相换算。2GB/3GB 也是 Google 的部署主张，不是独立手机实测的统一结果。</p>
<p>这一步的意义在于，竞争开始从“把模型做小”转向“让部署按档位可调”。同一套工具链里，开发者可以按设备和任务选择不同尺寸，而不是只拿到一个固定模型。但它也带来新的混淆：宣传页上的数字到底指哪一层，开发者要自己问清楚。</p>
<h3>免费的是调用，不是电</h3>
<p>Apple 的 WWDC25 开发者指南给了一句传播很广的话：“And it&#x27;s all available at no cost for any request.”（Apple Developer editorial team，2025-06-09，C05）。这句话成立的范围很窄：用户和开发者不按请求付平台调用费，而且请求可以离线跑。它没说电量、热预算、内存占用和设备折旧为零，也没说工作流不会另行调用云端。</p>
<p>独立研究正好补上这块。一篇 2025 年 7 月 2 日的移动 DVFS 预印本（arXiv 2507.02135 v1，上海交大与 Purdue 作者团队）在 Pixel 7 与 Pixel 7 Pro 等测试平台上，抽样了若干预填充和解码长度。论文报告：在相同能耗条件下，手机默认的 CPU、GPU、内存调频组合，时延最多比它测试的较优频率组合高 40.4%；作者的 FUSE 方案在 ShareGPT 评测里，TTFT 平均改善 7.0%–16.9%，TPOT 平均改善 25.4%–36.8%，并称每 token 能耗相同（C13）。</p>
<p>这些数字背后的机制，按论文自己的说法，是 CPU、GPU 和内存的调频器各自独立运行，可能选到次优频率，并且彼此影响。它说明的是特定设备、特定模型和采样负载下的运行时策略问题，不是“所有手机都会在提频后迟迟不回落”的普遍规律。省下云调用费，并不自动意味着用户设备上的计算成本、发热或交互延迟更低。本地请求只是把计算搬到了手机上，电量和时间仍要付。</p>
<h3>持续负载：两个页面给出两套结果</h3>
<p>持续负载更能说明问题。一篇 2026 年 6 月 7 日的跨平台预印本（arXiv 2603.23640 v2）正文称，用 Qwen 2.5 1.5B、4-bit 量化、固定 258-token prompt，四个平台在热机条件下每台设备重复 20 轮：iPhone 16 Pro 吞吐在前三轮下降约 40% 后稳定在约 23.7 tok/s，S24 Ultra 下降约 15% 后稳定在约 10 tok/s（C14）。</p>
<p>但同一版本的摘要却说，iPhone 16 Pro 在两轮内吞吐下降近一半，S24 Ultra 撞上 OS 强制的 GPU 频率下限而完全终止推理（C15）。同一篇论文的两个页面互相打架。这本身也是结论的一部分：单篇预印本、单次峰值，都不足以给手机排名。在作者更正、PDF 复核或独立复现之前，只能把两组描述并列，不能挑一组当成已稳定复现的事实，也不能推测哪一页更可信。</p>
<p>热这件事在手机上确实硬。在本文引用的测试设备上，手机靠被动散热，持续负载可能触发降频和限功耗；不同机型的散热设计不同，不能把这当成所有手机的统一行为。DVFS 预印本讲的是调频策略在相同能耗下跑出不同时延，持续负载预印本讲的是特定平台在长时间运行中的热行为，两者涉及不同因素，不能合并成同一个失效模式。它们共同支持的，只是一个更窄的判断：本地推理的瓶颈不只是算不算得动，还包括能不能一直算下去。</p>
<h3>标准开始搭台，但台子还很窄</h3>
<p>2026 年 6 月 15 日，MLCommons 在 MLPerf Mobile v6.0 里加了 Android 端侧 LLM 测试，模型选 Llama 3.2 1B/3B Instruct 和 Llama 3.1 8B Instruct，请求从 TinyMMLU、IFEval 里取；测试可在内存足够的设备上用 CPU 跑，8B 的 NPU 加速路径在 v6.0 发布说明里列出 Snapdragon 8 Elite Gen 5（C12）。</p>
<p>这是把移动 LLM 从厂商自选 benchmark 拉向固定任务的一步：模型、数据集、请求都定死，成绩才有可比性。但它仍是测试套件说明，不是成绩单；它覆盖 Android、覆盖指定加速路径，不覆盖 iOS 与 Android 同条件横比，也没有给出续航和持续热负载的完整结论。标准搭起了台子，台子还很窄。</p>
<h3>这台机器到底省了什么</h3>
<p>回到最初的问题。我的判断是：对确实留在本地的请求，端侧模型可以避免相应的云推理调用、一次网络往返，以及敏感输入离开设备的概率。它没有省掉计算，只是把计算从机房的电费，换成了你手机的电量、内存和热预算。至于现有产品实际把多少请求留在本地、总共省了多少，公开数据里没有本地请求占比，也没有本地与云端的可比总成本，所以无法做可靠的总量核算。</p>
<p>Apple 的 session 预算、Google 特定任务 API 的输出约束、Gemini Intelligence 的设备资格，是三个不同层级的限制。它们共同提示开发者：端侧任务要按上下文、输出形态和设备名单来规划；但不能说两家产品设计完全相同。对做音频或会议记录的产品来说，这意味着端侧更适合从已转写的文本里做短摘要、分类和实体抽取，而长稿、多轮上下文和复杂推理，仍要预留分段或服务器路径。</p>
<p>一个更窄的判断是：在持续负载场景，“每 Wh 能稳定处理多少真实请求”可能比峰值 TOPS 或参数量更能解释体验差异。这个判断仍是推测，需要匹配质量、任务和持续条件的独立测试来验证；DVFS 论文和那篇有内部冲突的持续负载预印本，只足以提示这个方向，不足以证明行业竞争重心已经转移。</p>
<p>一个可核查的预测是：到 2027 年底，主流手机平台仍会用短任务接口和设备资格来承接端侧模型，长上下文与强推理继续走服务器路径。如果届时主流平台普遍在本地处理长上下文且不需要服务器回退，这个判断就该重写。在那之前，把手机模型当成云端之前的本地执行层，比当成云端的替代品，更接近可查到的证据。</p></div>
</details>
