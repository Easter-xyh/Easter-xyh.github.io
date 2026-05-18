---
layout: post
title: "基于多智能体辩论的多模态情感识别一致性框架研究"
date: 2026-05-18
categories: 本科毕业论文 多模态情感识别
---

**🧠 用多智能体辩论解决多模态情感识别中的“意见不合”**

**📌 写在前面**

在现实对话中，人类情绪的传达从来不是单一的：语言说了什么、语气怎么说的、脸上是什么表情——这三者往往共同构成了我们对他人情绪的判断。

但在多模态情感识别任务中，这三类信息并不总是“步调一致”。比如：

文本说“我没事”，语气却很低落；

用户说“太棒了”，脸上却没有笑容；

音频中夹杂着背景笑声，模型误以为是说话人的情绪。

传统的多模态融合方法（如投票、加权、端到端融合）很难显式处理这种模态冲突，更缺乏可解释性。

于是，我提出了一套新框架：基于多智能体辩论的多模态情感识别一致性框架。

---

**🧩 核心思想：让三个专家“吵一架”**

我把情感识别任务变成一个多智能体辩论过程：

专家	模态	职责
📖 文本专家	文本	分析语义、情绪词汇、语气倾向
🎤 语音专家	音频	分析韵律、基频、能量、情绪强度
🎥 视觉专家	视频帧	分析面部表情、关键帧情绪线索
流程是这样的：
输入拆解：从视频中提取文本、音频、关键帧等；

首轮分析：每位专家独立输出情绪标签 + 置信度 + 证据；

多轮辩论：专家们依次发言，引用/质疑/补充其他专家的观点；

共识 or 融合：

如果达成共识 → 输出共识结果；

否则 → 加权融合（文本0.35，语音0.35，视觉0.30）。

**⚙️ 几个关键机制设计**

**🔒 证据锁定**

辩论过程中，专家不能随意“编造”新证据，只能基于首轮结构化结果进行解释和比较，避免大模型常见的“证据漂移”。

**🙅 弃权机制**

语音专家：检测到非说话人噪声（如笑声、背景音乐）或置信度低 → 弃权；

视觉专家：关键帧中无明显表情或置信度低 → 弃权；

文本专家：不弃权，语义弱时回落到 neutral。

弃权后，该专家在后续辩论中只输出固定语句，不再参与实质性讨论。

**🧠 共识判断**

只有当所有未弃权的专家输出相同格式的结构化共识信息时，才认为达成共识，否则进入加权融合。

**📊 实验设置与结果**

数据集：MELD（150条样本，7类情绪）

**模型配置：**

文本：qwen-plus

视觉：qwen3.6-plus

语音：qwen3-max + qwen3-omni

**🔍 冲突与纠偏**

**首轮冲突率**

54.67% 的样本中，三位专家首轮判断存在冲突
→ 说明模态不一致是常态，不是特例。

**弃权率**

61.33% 的样本中至少一个专家弃权
→ 弃权机制被频繁触发，说明其在真实场景中非常必要。

**纠偏能力**

情况	占比
文本首轮错误 → 最终正确	10.67%
文本首轮正确 → 最终错误	1.33%
多智能体辩论能有效修正单一模态的错误，同时不会显著破坏正确判断。

**🧭 优势和局限**

**✅ 优势**

模块化强，易于扩展和维护；

显式处理模态冲突，过程可解释；

弃权机制提升鲁棒性；

不依赖大规模训练，适合原型系统。

**❌ 局限**

实验规模偏小（150条）；

低频情绪（如 disgust）识别能力弱；

辩论质量受提示词和模型能力影响较大。

**🔭 未来方向**

扩大实验规模，完整测试 MELD；

优化提示词与专家工具；

引入动态发言顺序、自适应终止策略；

部署轻量级模型，降低推理成本；

扩展到更多场景：客服、教育、心理辅助等。

**📚 参考文献**

［1］Picard R W. Affective computing[M]. Cambridge: MIT Press, 2000.

［2］徐亚萍, 李艳燕, 李新, 等. 学习情感计算研究——基于国际研究的系统性文献综述[J]. 数字教育, 2024, 10(01): 1-9.

［3］Baltrušaitis T, Ahuja C, Morency L P. Multimodal machine learning: A survey and taxonomy[J]. IEEE transactions on pattern analysis and machine intelligence, 2018, 41(2): 423-443.

［4］何俊, 刘跃, 何忠文. 多模态情感识别研究进展[J]. 计算机应用研究, 2018, 35(11): 3201-3205.

［5］黄茂春. 基于视频数据的多模态情感计算研究[D]. 华南理工大学, 2023.

［6］Cheng Z, Cheng ZQ, He JY, et al. Emotion-llama: Multimodal emotion recognition and reasoning with instruction tuning[J]. Advances in Neural Information Processing Systems, 2024, 37: 110805-110853.

［7］Du Y, Li S, Torralba A, et al. Improving factuality and reasoning in language models through multiagent debate[C]//Forty-first International Conference on Machine Learning, Montreal, Canada, 2023: 1234-1245.

［8］Liang T, He Z, Jiao W, et al. Encouraging divergent thinking in large language models through multi-agent debate[C]//Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing, Abu Dhabi, UAE, 2024: 17889-17904.

［9］Fang C, Liang F, Li T, et al. Learning modality consistency and difference information with multitask learning for multimodal sentiment analysis[J]. Future Internet, 2024, 16(6): 213.

［10］Ortigoso A R, Vieira G, Fuentes D, et al. Project Riley: Multimodal Multi-Agent LLM Collaboration with Emotional Reasoning and Voting[EB/OL]. arXiv preprint, arXiv:2505.20521, 2025.  

［11］Wooldridge M. An Introduction to Multiagent Systems[M]. John Wiley & Sons, 2009.

［12］孟祥瑞, 杨文忠, 王婷. 基于图文融合的情感分析研究综述[J]. 计算机应用, 2021, 41(02): 307-317.

［13］Gerczuk M, Amiriparian S, Ottl S, et al. EmoNet: A transfer learning framework for multi-corpus speech emotion recognition[J]. IEEE Transactions on Affective Computing, 2023(2): 14.

［14］Luo Y, Wang S, Xu Z, et al. Confidence-Aware Self-Distillation for Multimodal Sentiment Analysis with Incomplete Modalities[C]//International Conference on Multimedia and Expo (ICME), Vancouver, Canada, 2025: 567-574.

［15］Ghosal D, Majumder N, Poria S, et al. Dialoguegcn: A graph convolutional neural network for emotion recognition in conversation[C]//Proceedings of the 2019 conference on empirical methods in natural language processing and the 9th International Joint Conference on Natural Language Processing (EMNLP-IJCNLP). 2019: 154-164.

［16］Hu D, Wei L, Huai X. Dialoguecrn: Contextual reasoning networks for emotion recognition in conversations[C]//Proceedings of the 59th Annual Meeting of the Association for Computational Linguistics and the 11th International Joint Conference on Natural Language Processing (Volume 1: Long Papers). 2021: 7042-7052.

［17］Hu J, Liu Y, Zhao J, et al. MMGCN: Multimodal fusion via deep graph convolution network for emotion recognition in conversation[C]//Proceedings of the 59th Annual Meeting of the Association for Computational Linguistics and the 11th International Joint Conference on Natural Language Processing (Volume 1: Long Papers). 2021: 5666-5675.

［18］Ghosal D, Majumder N, Gelbukh A, et al. Cosmic: Commonsense knowledge for emotion identification in conversations[C]//Findings of the association for computational linguistics: EMNLP 2020. 2020: 2470-2481.

［19］陶建华, 刘斌. 情感计算理论与方法[M]. 北京: 清华大学出版社.

［20］Argumentation in Artificial Intelligence[M]. Heidelberg: Springer, 2009.

［21］李太豪, 董建敏, 程翠萍, 朱敏. 情感计算：概念与原理[M]. 上海: 上海科学技术出版社.

［22］Tian L, Oviatt S, Muszynski M, et al. Applied Affective Computing[M]. New York: Association for Computing Machinery, 2022.

［23］Bansal R, Samanta B, Dalmia S, et al. LLM augmented LLMs: Expanding capabilities through composition[C]//The Twelfth International Conference on Learning Representations, Vienna, Austria, 2024: 101-110.

［24］Lei Y. In-depth study and application analysis of multimodal emotion recognition methods: Multidimensional fusion techniques based on vision, speech, and text[C]//Hainan International College, Communication University of China, Lingshui, China, 2024: 45-53.

［25］Majumder N, Poria S, Hazarika D, et al. DialogueRNN: An attentive RNN for emotion detection in conversations[J]. Proceedings of the AAAI Conference on Artificial Intelligence, 2019, 33: 6818-6825.

**💬 写在最后**

我不追求一次性打败所有 SOTA 模型，而是希望展示一种更透明、更可控、更贴近人类推理方式的多模态情感识别路径。

“吵一架，也许比闭眼投票更理性。”



