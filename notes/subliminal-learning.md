# 潜隐学习：模型如何通过“无意义数据”传递行为特征

**状态：** 旁支机制专题 / 持续更新  
**是否进入核心论文集：** 暂不进入。这个方向目前更适合作为机制争论与边界条件的阅读笔记，而不是核心证据链中的单一结论。

## 问题是什么？

所谓 **Subliminal Learning（潜隐学习）**，指教师模型带有某种行为特征（例如偏爱猫头鹰），只生成表面上与该特征无关的数据（如数字序列、代码或推理轨迹）；学生模型只在这些数据上训练，却可能继承教师的行为特征。

最值得追问的不是“数字里是不是偷偷写了猫头鹰”，而是：

1. 这种传递到底有多稳定、能否跨模型与跨任务复现？
2. 信号藏在什么地方——token 分布、局部 divergence tokens、激活方向、共享权重结构，还是训练上下文本身？
3. 为什么相同或高度相关的 base model 更容易发生传递，而不同 base model 往往不行？
4. 不同实验看到的机制，究竟是同一个过程的不同切面，还是多个不同机制被统称为“潜隐学习”？

## 最短结论

目前较稳妥的说法是：

> **潜隐学习是真实、可复现但具有明显边界条件的模型蒸馏现象。** 教师特征可以通过人类看来无语义的数据留下微弱统计或梯度信号，并在合适的学生模型与训练设置中重新形成相关行为；但现有研究尚未支持一个能够解释所有实验设置的统一机制。

尤其要避免把以下几件事混为一谈：

- token 的静态几何关系；
- 隐藏状态中是否能读出某个概念；
- 某个隐藏状态是否因果控制最终行为；
- 训练时学生为什么真正继承教师特征。

这些是不同层级的问题。

---

## 阅读地图

### 1. 原始现象：Cloud et al. — Subliminal Learning

- **论文：** [Subliminal Learning: Language models transmit behavioral traits via hidden signals in data](https://arxiv.org/abs/2507.14805)
- **正式发表：** [Language models transmit behavioural traits through hidden signals in data, Nature (2026)](https://www.nature.com/articles/s41586-026-10319-8)

**作用：** 建立现象本身。

教师被赋予某个 trait 后，只生成数字、代码或推理数据，学生仍可能继承该 trait；过滤掉显式语义线索也不能完全消除效应。论文同时发现，不同 base model 的师生通常不会发生同样的传递，并给出一个一般性的神经网络理论结果：在特定条件下，共享初始化会让学生模仿教师输出时的参数更新与教师更新方向产生系统性对齐。

**当前意义：** 它告诉我们“现象存在”，但没有独自解决“LLM 中具体靠什么机制发生”。

---

### 2. Token Entanglement：数字与概念 token 会不会天然纠缠？

- **作者项目页：** [It's Owl in the Numbers: Token Entanglement in Subliminal Learning](https://owls.baulab.info/)

**作用：** 提出早期、直观的 token-level 解释。

研究发现某些概念与数字 token 的 logits/输出方向会出现耦合，例如提高某个数字 token 的概率也可能提高 “owl” 的偏好；甚至只在 prompt 中放入相关数字，就能影响模型偏好。

**限制：** 后续工作表明，全局 token entanglement 不是潜隐学习发生的必要条件，因此它更像一个真实存在的局部机制或相关现象，而不是统一解释。

---

### 3. Divergence Tokens：真正携带信号的可能只是少数位置

- **论文：** [Towards Understanding Subliminal Learning: When and How Hidden Biases Transfer](https://arxiv.org/abs/2509.23886)

**作用：** 把注意力从“整个数字分布”缩到少数关键 token。

研究发现，有偏教师与对照教师真正出现预测差异的少数 **divergence tokens** 对传递非常关键；遮掉这些位置后，隐藏偏向大幅减弱。它还指出早期网络层尤其重要，甚至只微调一个早期层也可能产生潜隐学习。

同时，这项工作显示现象相当脆弱：轻微改写 prompt 就可能显著抑制传递。

**当前意义：** 说明信号可能非常稀疏，而且依赖具体训练上下文。

---

### 4. LoRA / Context 边界：潜隐学习会不会只是某些训练设置的产物？

- **论文：** [Subliminal Learning is a LoRA Artifact](https://arxiv.org/abs/2606.00831)

**作用：** 提供强反方与边界条件。

该工作报告：传递强度与 LoRA rank 呈明显依赖，在其设置中 full finetuning 会使现象消失；训练和评估时共享的 system prompt、chat template 等上下文也非常关键。作者因此主张，至少他们观察到的大量效应更像是 LoRA 与上下文共同形成的脆弱通道，而不是稳定、普适的行为传递机制。

**为什么必须保留：** 即使后续研究不同意“只是 artifact”这一最强结论，它仍提醒我们不能把任一实验配置里的效应直接推广成模型蒸馏的普遍规律。

---

### 5. Steering Vector Distillation：微弱梯度是否沿着同一个内部方向累积？

- **论文：** [Subliminal Learning Is Steering Vector Distillation](https://arxiv.org/abs/2606.00995)

**作用：** 给出一个很有解释力的机制模型。

在作者研究的两类开源模型中，教师的 system-prompt trait 可以近似为一个 steering vector；学生在教师输出上训练时，会逐渐学到与这条向量对齐的内部方向。研究还发现，自适应优化器有助于把非常小但方向一致的梯度分量长期积累起来，而非自适应优化器更容易让它被大梯度噪声淹没。

它也给出一种“为什么不同 base model 之间通常不传”的解释：同一个语义 steering 方向会在不同模型中产生不同的模型特异副作用，因此教师数据里的非语义信号不一定能被另一个坐标系的模型还原成同一 trait。

**限制：** 这一解释对某些 prompt/trait 很漂亮，但不能自动推广到所有潜隐学习现象。

---

### 6. Non-Semantic Distillation：同样的行为，不一定对应同一种内部实现

- **论文：** [Subliminal Learning is Non-Semantic Distillation](https://arxiv.org/abs/2608.05734)

**作用：** 给 steering-vector 解释增加重要限制。

作者发现，给师生权重加入高斯噪声反而可以增强部分潜隐传递，提示模型特有的非语义权重结构具有重要作用。更关键的是：由 steering vector 诱导出的教师，会让学生继承 steering-like 的内部结构；而通过 prompt 诱导出的教师，学生并不一定形成同样的内部形式。

**当前意义：** “学生最后表现得像教师”不意味着学生内部必然复制了教师的同一种机制。潜隐学习可能是一类现象，而不是一条单一通路。

---

### 7. 独立复现与泛化边界

- **论文：** [Reproducing and Evaluating the Generalizability of Subliminal Learning in Open-Weight Models](https://arxiv.org/abs/2609.12586)

**作用：** 检查原始效应能否在开放权重模型中重现并扩展。

该复现总体支持原始现象，但扩展实验显示传递并不普遍：强度随 trait、任务和模型显著变化，其中一个模型几乎没有效应。

**当前意义：** “现象存在”与“现象普遍”是两件不同的事。任何总结都应保留这种异质性。

---

### 8. Causal Depth：不要把静态几何、可读性与因果控制混在一起

- **论文：** [Subliminal Prompting Beyond Static Geometry: Causal Depth and Multi-Token Confounds](https://arxiv.org/abs/2609.19149)

**作用：** 澄清几个常被混用的测量层级。

研究在 frozen prompting 设置中分别测量：固定输出向量几何、隐藏状态可读性与因果控制。它发现，从 Llama-3.1-8B 到 70B，静态输出向量几何对行为的预测反而变差，但把 donor prompt 的中间隐藏状态移植到 recipient 后，donor 对最终答案的因果控制显著增强，且在大模型中更早形成。

在 Qwen 的多 token 数字分析中，简单 per-token averaging 还会制造一个控制数字长度后消失的正相关，说明测量方式本身可以产生假象。

**边界：** 这项工作研究的是 frozen prompting channel，而不是训练时 trait transfer；它能排除一些过于简单的 token-level 解释，却不能直接宣布潜隐学习的训练机制是什么。

---

## 当前可以拼出的工作模型

一个暂时兼容多篇研究、但仍需继续检验的图景是：

1. 教师的 trait 改变内部激活与输出概率分布；
2. 这种改变在人类看来无语义的数据里留下非常微弱、模型特异的统计差异；
3. 差异可能集中在少数 divergence tokens、共享上下文位置或某些激活方向；
4. 当学生与教师拥有足够匹配的 base-model 几何、初始化或函数结构时，这些差异产生的训练梯度会在某些方向上稳定积累；
5. 优化方式、LoRA rank、prompt/chat template、trait 类型与任务都会决定这个信号能否被放大到可观察行为；
6. 学生最终得到的内部实现未必与教师完全相同，因此“同一行为”不应被当作“同一内部机制”的证明。

这套图景比“数字里藏着语义暗号”更符合现有证据，但它仍不是闭合理论。

## 仍未解决的问题

- LoRA/context 依赖究竟解释了多少原始结果？在系统控制的 full finetuning 下，哪些 trait 仍能稳定传递？
- steering-vector distillation 是主机制、一个常见特例，还是只对某类可线性表示的 trait 成立？
- divergence tokens、shared-init 梯度对齐与 steering-vector 学习之间能否被统一到同一个定量理论中？
- 为什么某些模型、trait 和任务几乎不传，而另一些十分明显？
- “同 base model 更容易传递”究竟需要多强的参数/表征对齐？同架构、同预训练谱系、同初始化分别贡献多少？
- frozen subliminal prompting 与 training-time subliminal learning 到底共享多少机制？

## 对核心论文集的处理建议

**暂不新增核心条目。**

理由不是这组研究不重要，而是它目前最有价值的地方恰恰在于“机制尚在竞争”。把其中任一篇压缩成核心论文集的一条稳定结论，都会丢失大量边界条件。

如果未来出现：

- 跨实验室、跨训练设置都能复现的统一机制；或
- 明确解决“同 base model / 不同 base model 为什么差异巨大”的机制性工作；或
- 能把 divergence tokens、gradient alignment、steering direction 与训练上下文统一起来的更强研究，

再考虑是否让这一支进入核心手册。

---

> **一句话记忆：** 潜隐学习不是“模型用数字写秘密暗号”这么简单；它更像教师在自己的参数与概率地形里留下微弱脚印，而只有某些拥有相似地形、并在合适训练条件下行走的学生，才会沿着这些脚印重新走向相似的行为。
