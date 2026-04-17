# LLM as a Judge：研究共识、典型失效模式与工程设计启发

---

## 1. Introduction

LLM as a Judge 已经从一种研究实践，逐步变成生成式系统开发中的基础设施组件。它被用于开放式文本质量评估、RAG 忠实度检查、A/B 对比、数据筛选、偏好标注，以及多模态场景中的自动化评测。其吸引力很直接：传统词面指标难以覆盖开放式任务，而纯人工评测又昂贵、缓慢且难以持续运行。[1][2]

本文区分研究共识、经验性观察与作者推断。

本文的核心判断是：**LLM judge 的主要价值，不在于替代人类裁决，而在于把开放式质量评估程序化；在开放式、弱约束评测中，其主要风险往往来自系统性地依赖错误代理变量来近似"质量"。** 这意味着，LLM judge 更适合被理解为一个有偏、可约束、可校准的评测组件，而不是一个天然可靠的真值机。[1][2][5]

本文不试图回答“哪一个 judge 最强”，而是聚焦四个更基础的问题：

1. LLM judge 当前有哪些相对稳定的研究共识；
2. 它为什么会出现位置偏差、长度偏差、自偏好等系统性问题；
3. pairwise、reference/evidence、结构化评测分别在缓解什么；
4. 如果把 judge 接入真实系统，哪些设计原则最值得优先遵守。

---

## 2. Background and Related Work

### 2.1 什么是 LLM as a Judge

LLM as a Judge 指用一个语言模型去评估另一个模型或系统输出的质量。评测对象可以是单条文本、成对候选答案、长对话、RAG 响应、代码输出，甚至多模态生成结果。与 BLEU、ROUGE、BERTScore 这类静态指标相比，LLM judge 的优势在于能够处理更开放的语义目标，例如帮助性、忠实度、完整性、风格适配性和复杂指令遵循。[1][3]

更准确地说，LLM judge 不是一个单一方法，而是一类**评测协议**：底层模型、提示模板、是否给 reference、是否做 pairwise、是否要求解释、是否做重复采样、是否有人类校准，这些因素共同决定了 judge 的行为与可靠性。[1][2]

### 2.2 主流评测范式

现有工作大致可分为五类。

**Absolute scoring**：对单个输出直接给分，通常配合 rubric 或 form-filling。G-Eval 是这一方向的代表，它通过 CoT 与表单化评分改进 NLG 评测与人类判断的一致性。[3]

**Pairwise comparison**：让 judge 在 A/B 两个候选之间做偏好判断。这类方法通常比绝对打分更稳定，也更接近人类相对比较的习惯，因此在竞技场式评测、模型版本对比和偏好数据构造中被广泛采用。[1]

**Rubric-based judging**：把抽象目标拆成多条显式标准，例如 factuality、relevance、instruction following、tone。Prometheus 2 这类专门的开源 judge 模型强调的就是“可按用户自定义标准执行评测”。[4]

**Reference / evidence-based judging**：向 judge 提供参考答案、检索证据或规则文本，以减少仅凭语言表象作判断的风险。OpenAI 的官方评测建议明确强调：高质量 eval 依赖清晰标准、代表性数据和可靠参考材料。[2][5]

**Committee / structured judging**：通过多 judge、多轮验证、结构化问题分解或显式投票来提升稳定性。这类方案在复杂长答案评测和高风险场景中越来越常见，但也会带来成本、时延和协议复杂度的上升。[1]

### 2.3 当前较稳定的共识

**稳定共识 1**：LLM judge 不是通用真值替代物，而是一个需要协议设计、校准和人工锚定的评测器。现有 survey 与官方实践文档都强调了这一点。[1][2]

**稳定共识 2**：Pairwise 往往比单次绝对打分更稳，但它并没有消除偏差，只是把问题转移到了顺序敏感性和比较协议设计上。[1][6]

**稳定共识 3**：在 correctness-sensitive 场景中，reference/evidence 往往比单纯更换更强 judge 更能改善与人工判断的一致性。没有外部锚点时，judge 很容易把 plausibility 当 correctness。[2][7]

**稳定共识 4**：judge 的可靠性不仅取决于模型能力，还取决于评测目标是否被显式定义。若“好答案”的质量函数本身模糊，judge 往往会用表面特征填补决策空缺。[1][2]

---

## 3. Problem Formulation

本文把 LLM judge 的核心问题表述为一个**测量错位**：

> 我们希望 judge 承担的是“稳定、可比较、可复核的质量测量”；但底层模型实际上是一个“顺序敏感、分布驱动、逐 token 生成判决文本”的生成系统。

这个错位至少包含三层。

### 3.1 目标错位

评测目标往往并不单一。所谓“一个答案好不好”，通常混合了 factuality、helpfulness、coverage、brevity、style、safety、instruction following 等多个维度。若这些维度没有被显式写清，judge 就会隐式地自己分配权重，而这些权重可能随 prompt、模型和数据分布改变。[1][2]

### 3.2 比较错位

很多评测任务实际上要求一种**对称比较器**：交换 A/B 顺序后，结论原则上不该改变；尺度稳定时，同一个“7 分”应在不同样本中具有相近含义。但 LLM 的原生训练目标并没有保证这些测量学性质。[1][6]

### 3.3 证据错位

在没有 reference/evidence 的场景里，judge 只能依赖内部语言分布和表面风格信号。此时它测的常常不是 correctness，而是“像不像一个高质量答案”。No Free Labels 对这一点给出了较直接的证据：judge 对某个问题的评判能力，与它自己能否回答该问题高度相关；而提供高质量人类参考答案，可以明显改善其与人类标注的一致性。[7]

### 3.4 一个更具体的工作定义

从工程视角看，LLM judge 更接近下述对象：

> 一个在给定协议、给定材料、给定输出约束下，对候选输出做近似判别的生成模型。

这一定义有两个直接含义。第一，judge 的性质是**协议相关**的，而不是模型内生的。第二，judge 的输出应被理解为**条件化测量结果**，而不是无条件真值。

下文讨论的许多偏差，应理解为协议、材料与模型共同作用的结果。

---

## 4. Analysis and Comparison

### 4.1 位置偏差：为什么 pairwise 不是天然对称的

**稳定共识**：Pairwise 判断经常受到答案顺序影响。[1][6]

**代表性发现**：A Systematic Study of Position Bias in LLM-as-a-Judge 系统分析了 pairwise 与 listwise 场景中的位置偏差，并引入了 repetition stability、position consistency、preference fairness 等指标来量化这一问题。[6]

**文献直接支持的事实**：position bias 不是随机噪声。已有研究表明，其严重程度受质量差距、任务类型、judge 模型等因素影响——当两个候选项质量接近时，顺序扰动更容易直接翻转最终结论。[6]

**可能解释**：一个合理解释是，judge 并不是先独立估计 A 与 B 的绝对质量，再做一个显式减法；更常见的情况是，它在给定顺序的上下文中直接生成 verdict。这样一来，候选项顺序会改变比较时的上下文表征与局部锚定路径。结合长上下文中已被广泛观察到的位置敏感现象，这使得顺序翻转在工程上应被视为 judge 协议的基础体检，而不是可选增强项。

**作者推断**：从更底层看，这意味着自回归 LLM 天然不是一个 permutation-invariant comparator。位置翻转测试应被视为 judge 协议的基础体检，而不是可选增强项。

### 4.2 长度偏差与风格偏差：为什么“像好答案”会被误判成“就是好答案”

**稳定共识**：LLM judge 容易偏爱更长、更完整、格式更规范、语气更稳的回答。[1] 多项工作发现，这一现象在 MT-Bench、Chatbot Arena 等评测中被系统观测到。[20]

**文献直接支持的事实**：长度偏差与风格偏差通常来自训练分布中的代理变量学习。人类偏好标注数据显示，当事实准确度难以快速验证时，标注者会不由自主地偏好字数更多、细节更丰富的答案，这种"冗长偏好（Verbosity Bias）"被奖励模型（RM）捕捉并放大。[14][15] 对齐算法（PPO、DPO）会进一步放大这一偏差：PPO 依赖已内化长度偏差的 RM 进行策略更新，容易通过增加输出长度骗取更高奖励；DPO 直接用交叉熵拟合偏好数据，研究表明其对长度等表面特征更易过拟合。[16][17]

**可能解释**：一个合理解释是，模型将"长度"和"风格规整度"学成了 quality proxy。当真实质量函数不清晰或差异很小时，模型会用这些廉价且稳定的表面特征来补足决策。

**作者推断**：长度偏差与位置偏差本质上属于同一类问题：当真实质量函数不清晰或差异很小时，模型会用廉价且稳定的表面特征来补足决策。此外，当 Prompt 要求 Judge 综合评估多个维度并给出总分时，模型可能发生决策降级——用容易计算的风格/长度分数去掩盖或替代难以计算的事实分数。[20]

### 4.3 自偏好与分布熟悉度：为什么 judge 会偏爱"更像自己会写的答案"

**代表性发现**：自偏好相关研究表明，judge 倾向于偏爱自己生成或更符合自身风格分布的答案。[8][9][20][21] Zheng 等人 [20] 在 MT-Bench 和 Chatbot Arena 的实验中系统量化了这一现象，并将其命名为"自我增强偏差（Self-enhancement bias）"；Panickssery 等人 [21] 进一步揭示了模型血统（Lineage Bias）如何通过微调过程被遗传和放大。

**文献直接支持的事实**：评测模型与其评估对象在训练数据集上的重叠度越高，打分时的正向偏见就越强烈。大量开源模型在 SFT 和 RLHF 阶段使用了由 GPT-4 或 Claude 蒸馏出的合成数据，这些模型作为 Judge 时会表现出强烈的"GPT-4 偏好"。[21]

**可能解释**：一个合理解释是，模型将"分布熟悉度（Distributional Familiarity）"作为了质量评分的代理标尺。当 LLM 评估自己（或同架构模型）生成的文本时，该文本在其内部概率空间中具有更高的似然度，模型会本能地倾向于选择"更不意外"的文本。Wataoka 等人 [8] 从理论上验证了这一假说；Chen 等人 [9] 则指出，候选文本的风格特征向量与评测模型自身的特征空间高度重合时，模型会被这种底层特征的重合度所"诱导"。

**作者推断**：在没有外部参考锚点的开放式评测中，LLM Judge 的评估更容易受到分布熟悉度的影响。它可能告诉你"这个答案看起来像不像我会给出的好答案"，但未必能准确判断"这个答案是否真的符合客观事实"。这也是为什么在没有 reference 的场景里，judge 容易奖励流畅但错误的答案。

### 4.4 没有 reference 时，judge 到底在测什么

**代表性发现**：No Free Labels 研究表明，LLM judge 在"判对错"任务上的表现与其自身解决该问题的能力高度相关；更重要的是，提供高质量人类参考答案，往往比使用更强的 judge 模型但搭配较差的参考材料效果更好。[7]

**机制解释**：缺乏 reference/evidence 时，judge 实际执行的是 plausibility assessment（似真性评估），而非 correctness assessment（正确性评估）。它能判断"这回答是否像一个可信回答"，但未必有能力判断"这回答是否真的正确"。这解释了为何许多 RAG 或专业 QA 系统，即便使用强 judge，仍会出现高分幻觉。[2][7]

以下从三个层面深入剖析这种事实性漂移的底层机制：

**（1）似真性（Plausibility）对正确性（Correctness）的僭越**

在无外部锚点时，LLM 评估事实性的本质是计算文本在预训练知识中的"连贯性"与"似然度"。如果候选答案包含生造术语或虚假引用，只要构词法符合语言规律且格式规范，文本的局部交叉熵就会很低。Judge 模型没有真实世界校验引擎，它通过注意力机制感知到文本内部上下文的强关联（如虚假前提后的严密推导），就会赋予高分。

Min 等人 [22] 在 *FActScore* 中表明，即使是最强模型（如 GPT-4），在评估长文本事实性时，如果没有将其拆解为原子事实并与外部知识库对齐，其打分会严重偏向那些"流畅但包含幻觉"的连贯文本。Krumdick 等人 [7] 也证实，在缺乏外部 Grounding 时，Judge 实际执行的是似真性评估（Plausibility Assessment）而非正确性评估。

**（2）参数化记忆的长尾塌缩（Long-tail Knowledge Collapse）**

LLM 的知识以参数形式压缩，这种压缩是有损的且对训练数据曝光频率高度敏感。对于高频知识，模型的参数化记忆相对稳固；但对于长尾知识、垂直领域专有事实或近期动态，参数化记忆则非常模糊。当 Judge 面对这类领域的错误候选答案时，由于自身提取不出正确的知识表征，它便无法产生"预测冲突（Prediction Conflict）"，判决权被移交给了语言风格和长度特征。

Mallen 等人 [23] 系统分析了闭卷状态下模型的知识召回率，研究表明面对处于知识分布尾部的问题，模型参数化记忆的可靠性呈断崖式下跌。这意味着将大模型直接作为领域知识的"无参考裁判"在底层机制上存在根本局限。

**（3）评判能力的天花板边界（Capability Bound）**

在判定任务中，Judge 的能力上限被其自身的生成能力严格锁死。在自回归模型中，判断与生成共用同一套表征空间。如果模型自身无法解决某个复杂问题或推导出某项事实，它就无法在内部表征空间中重构出正确的逻辑链，因此也无法识别出候选答案在关键推理步骤上的谬误，只能诉诸表面特征。

Bowman 等人 [24] 在关于可扩展监督（Scalable Oversight）的研究中揭示，一旦任务难度超过了模型（作为 Evaluator）自身的知识边界，即使给予再多的 Prompt 提示，其提供的监督信号也会彻底失效并沦为噪音。

**阶段性结论**：在 correctness-sensitive 任务中，无参考评估显著退化为 plausibility assessment。

### 4.5 全局评估与原子分解：两种不同的评测假设

**全局评估**假设模型可以在整体语境中直接形成高层判断。它的优点是保留全局耦合信息，尤其适合整体连贯性、风格统一性、用户体验等强交互目标；缺点是维度混合严重、可诊断性弱、在高风险任务里不够可控。[1][3]

**原子分解再聚合**假设整体质量可以拆成若干局部判断后再加权。它的优点是可控、可审计、适合 error localization，也更容易结合规则、reference 和 evidence；缺点是分解未必正确，局部项可能高度相关，线性加权也可能掩盖跨项交互。[10]

**代表性发现**：DeCE 这类分解式方法把评测拆成 precision 与 recall 两条轴，展示了结构化拆分在高风险长答案评测中的可解释性收益。[10]

**作者推断**：全局与分解不是简单的优劣关系，而是两种不同假设。前者更像感知整体质量，后者更像编译评估规则。真实系统里更稳的方案通常是混合式：高风险、强规则维度原子化，整体体验维度保留全局判断。

### 4.6 推理策略与 judge 稳定性：temperature、top-p、top-k 为什么也会影响评测

**稳定共识**：生成式评测天然存在方差，单次调用不能直接等同于真值。[2]

**工程启发**：由于生成式评测天然存在方差，judge 通常更适合低随机性、结构化输出以及必要时的重复采样协议。至于是否采用先分析再给 verdict 的两阶段输出，应视任务类型和成本约束做实验验证，而不应预设其必然更优。

### 4.7 一个统一解释：judge 的问题为什么会成系统地重复出现

综合上面的现象，可以把 LLM judge 的常见问题归因到三个更深层的错位：

1. **生成系统被拿来做测量。** LLM 原生是条件生成器，不是标定良好的比较器。
2. **分布熟悉度被误当成质量。** 训练中最容易利用的表面规律，往往会成为 judge 的 shortcut。
3. **含混目标被压缩成单次决策。** 当质量函数没有被清晰写出来时，judge 只能靠启发式近似。

这三点并不保证任何具体样本一定出错，但它们解释了为什么位置偏差、长度偏差、自偏好、无 reference 失真会在不同任务中反复出现。

---

## 5. Discussion and Implications

### 5.1 工程定位与功能边界

在工程实践中，LLM judge 应被理解为**评测系统中的一个组件，而非终审者**。OpenAI 的官方文档将 evals 明确定位为测试与改进生产系统的手段，强调需先建立清晰标准、代表性数据和人工校准流程。[2][5] 这一定位包含三层核心认知。

首先，**judge 不是通用真值替代物**。如第 4 章所述，LLM judge 存在位置偏差、长度偏差、自偏好、无参考时的事实性漂移等系统性问题。它无法像人工专家那样进行基于外部知识核验的 correctness assessment，而只能执行基于内部概率分布的 plausibility assessment。其次，**关键设计对象是评测协议，而非模型本身**。同一个底层模型，在不同 prompt、reference、顺序控制和输出约束下，可靠性差异可以非常大。[1][2] 盲目追求"更强模型"而忽视协议设计，是工程实践中的常见误区。再次，**优化的目标是评测 pipeline 的整体可控性，而非单条样本的 judge 智能**，这包括数据集设计的代表性、pairwise 比较时的顺序翻转、reference/evidence 的提供方式、错误归因与人工复核路径的完整性。

在功能边界上，LLM judge 最适合承担偏好比较（在质量差异明显的候选之间做相对排序，需配合顺序翻转）、体验评估（对帮助性、风格、可读性等主观维度进行全局判断）以及初筛与粗排（在大规模样本中快速过滤明显低质内容）。相反，在缺乏外部 reference/evidence 时，不宜单独承担高风险事实性终审（尤其是长尾知识领域）、单一真值判定（将开放式回答简单二值化为正确/错误）以及闭环训练反馈（同时作为评估者和训练信号源易导致 reward hacking）。

### 5.2 六条工程建议

基于第 4 章对各类偏差机制的分析，以下按优先级给出六条工程建议。

**【强建议】第一，先定义质量函数，再选择 judge 范式。** 这是最关键的前置步骤。4.1 节的位置偏差、4.2 节的长度/风格偏差、4.3 节的自偏好，其共同根源在于"含混目标被压缩成单次决策"。当质量函数本身模糊时，judge 只能用表面特征（顺序、长度、熟悉度）填补决策空缺。[1][2] 因此，在选定 judge 模型前，必须先明确回答质量的显式维度（如 factuality、helpfulness、coverage、brevity 等）及其权重。目标模糊时，换更强模型通常无济于事。此外，针对 4.3 节的自偏好现象，生产系统中应严禁使用与被评测系统相同架构或蒸馏血统的模型作为唯一 Judge，并在高风险评估链路中引入异构模型委员会（如 GPT-4 + Claude-3.5 + 专有小参数模型）进行多数投票。Panickssery 等人 [21] 证实了模型血统导致的正向偏见；Chen 等人 [9] 从隐式风格签名的角度解释了该机制。

**【强建议】第二，在 correctness-sensitive 场景中优先采用 reference/evidence-based judge。** 针对 4.4 节"无参考时的事实性漂移"问题——模型无法区分"符合逻辑的陈述"与"符合事实的陈述"，只能执行 plausibility assessment。[2][7] 现有研究反复表明，高质量 reference 往往比单纯更换更强 judge 更能改善与人工标注的一致性。生产系统中，事实类评测应强依赖 RAG 提供证据字典，并在 Prompt 中强制约束 Judge 仅做事实核对，将复杂的质量判定降维成"蕴含/矛盾"二分类任务。没有外部锚点时，高分常常只是高 plausibility，而非高 correctness。Krumdick 等人 [7] 明确区分了 plausibility 与 correctness；Min 等人 [22] 确立了事实性评测必须依赖外部知识库锚定的工程基线。

**【建议】第三，模型对比宜优先采用 pairwise，但建议配套顺序翻转。** 针对 4.1 节的"位置偏差"——自回归 LLM 的单向自回归架构与上下文注意力机制天然破坏了比较任务的置换不变性，候选项的先后位置会改变上下文表示和注意力分配。[6] 生产系统中必须对每一次 Pairwise 比较执行强制翻转，即同时运行 A vs B 与 B vs A。若两次结果一致则采纳结论；若发生翻转，应将该样本视为不稳定样本，而不是直接采纳任一单次 verdict。Wang 等人 [11] 系统论证了单向注意力对评估公平性的破坏，确立了多位置采样的工程共识；Zheng 等人 [20] 也证实了位置翻转作为防抖动基线的重要性。

**【建议】第四，将 hard constraints 与 soft preferences 分离。** 针对 4.2 节的"长度/风格偏差"和 4.4 节的"似真性僭越"——模型容易用风格/长度分数掩盖难以计算的事实分数，将 hard failure（事实错误、规则违规）与 soft preference（风格、自然度）混为一谈。人类偏好标注数据显示，当事实准确度难以快速验证时，标注者会不由自主地偏好字数更多、细节更丰富的答案，这种"冗长偏好"被奖励模型捕捉并放大。[14][15] 生产系统中，事实错误、规则违规、证据缺失应设置硬门槛（hard gate），一旦触发直接拒收；风格、自然度等主观维度单独评估，不与事实性在同一线性总分里混算。同时应在输入 Judge 前通过预处理脚本抹除排版标签并截断/填充至相近长度，强迫注意力机制聚焦语义。

**【建议】第五，优先要求 judge 输出结构化 verdict 与 failure reason，而不是单个总分。** 针对 4.2 节的"维度混合导致的决策降级"——单分值常掩盖真实错误结构，无法定位具体失败维度。[2][10] 应构建多级评估 DAG 管道替代单一总分：L0 门控层用正则/代码脚本拦截字数、格式等确定性错误；L1 红线层专注于安全性和事实准确性的判定，赋予"一票否决权"；L2 体验层仅对通过 L1 的答案进行 Pairwise 流畅度/风格对比。要求 judge 输出结构化判断（如各维度分项评分 + 失败原因）有助于 error localization 和针对性改进。Yu 等人 [10] 验证了结构化拆分在捕捉高风险错误中的可解释性收益。

**【建议】第六，把 judge 当成需要被校准的对象。** 针对 4.6 节的"推理策略与稳定性"——生成式评测天然存在方差，单次调用不能直接等同于真值。[2] Temperature、Top-p 等采样参数会引入随机性扰动，在决策边界时极易导致 verdict token 的翻转。生产链路的 Judge 接口应强制设置 Temperature = 0，全面接入 Structured Outputs，通过在 Logit 层面对解码树进行语法裁剪来收敛内部方差。上线前还应先做人类一致性检查（human agreement study），量化 judge 与人工标注的一致性水平；必要时采用重复采样或多数投票降低方差。OpenAI [2] 明确建议使用 0 temperature 和强制结构化输出；Wang 等人 [25] 解释了通过生成推理路径来稳定最终答案概率分布的底层机制。

在实践中还需警惕以下反模式：盲信单次 judge 输出（单次 verdict 只是样本，不是稳定测量）、用总分掩盖 hard failure（风格很好但事实错误的答案不应因总分"还不错"而被放过）、闭环污染（让同一个 judge 同时定义标准、筛数据、做训练反馈、验收上线，会激励模型学会"讨好 judge"而非服务真实用户），以及 benchmark 外推谬误（不同任务、数据分布、reference 条件下，judge 行为可能显著变化，不可直接将文献中的相关性外推至自身业务）。

### 5.3 总结

本文的核心论点是：**LLM judge 的主要价值不在于替代人类裁决，而在于把开放式质量评估程序化；其主要风险也不在于偶尔误判，而在于系统性地依赖错误代理变量来近似"质量"**。从第 4 章的机制分析到第 5 章的工程原则，贯穿始终的是对"测量错位"的认知：LLM 原生是条件生成器，却被要求承担稳定、可对称、可标定的测量任务。这一错位解释了位置偏差、长度偏差、自偏好、无参考时的事实性漂移为何会在不同任务中反复出现。

对工程团队而言，最可靠的方向不是盲目追求更强 judge，而是遵循"先定义目标、再设计协议、最后选择模型"的三步路线。在当前阶段，LLM judge 最适合作为**有偏、可约束、可校准的评测组件**接入生产系统，而非作为**无偏、通用、自洽的真值机**。只有在这一认知前提下，才能充分发挥其工程价值，同时规避系统性风险。

---

## 6. Conclusion

LLM as a Judge 的核心价值，是把开放式质量评估从“完全依赖人工”推进到“可工程化、可持续运行、可部分自动化”的阶段。[1][2]

但从理论上看，LLM judge 天然存在一个难以回避的错位：它是一个顺序敏感、分布驱动、逐 token 生成判决的系统，却被要求承担稳定、可对称、可标定的测量任务。这一错位解释了位置偏差、长度偏差、自偏好、无 reference 时的事实性失真，以及训练闭环中的 reward hacking 风险。[1][6][7][8][9]

因此，对 LLM judge 最合理的理解不是“它能不能替代人”，而是“它在什么协议下可以成为一个足够有用的近似评测器”。在当前阶段，真正可靠的方向不是盲目追求更强 judge，而是继续改进评测协议：更清晰的质量定义、更强的 reference/evidence 约束、更合理的结构化拆分、更严格的校准与人工兜底。[1][2][7][10]

对工程团队而言，这意味着一条很具体的路线：**先把评测目标写清楚，再把评测协议设计清楚，最后才讨论用哪个 judge。**

---

## References

[1] Gu, J. et al. *A Survey on LLM-as-a-Judge*. arXiv:2411.15594, 2024. https://arxiv.org/abs/2411.15594

[2] OpenAI. *Evaluation best practices*. Official documentation, accessed 2026-04-16. https://developers.openai.com/api/docs/guides/evaluation-best-practices

[3] Liu, Y. et al. *G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment*. EMNLP 2023. https://aclanthology.org/2023.emnlp-main.153/

[4] Kim, S. et al. *Prometheus 2: An Open Source Language Model Specialized in Evaluating Other Language Models*. arXiv:2405.01535, 2024. https://arxiv.org/abs/2405.01535

[5] OpenAI. *Graders*. Official documentation, accessed 2026-04-16. https://developers.openai.com/api/docs/guides/graders

[6] Shi, L. et al. *A Systematic Study of Position Bias in LLM-as-a-Judge*. IJCNLP 2025. https://aclanthology.org/2025.ijcnlp-long.18/

[7] Krumdick, M. et al. *No Free Labels: Limitations of LLM-as-a-Judge Without Human Grounding*. arXiv:2503.05061, 2025. https://arxiv.org/abs/2503.05061

[8] Wataoka, K. et al. *Self-Preference Bias in LLM-as-a-Judge*. OpenReview, 2024. https://openreview.net/forum?id=Ns8zGZ0lmM

[9] Chen, Z.-Y. et al. *Beyond the Surface: Measuring Self-Preference in LLM Judgments*. EMNLP 2025. https://aclanthology.org/2025.emnlp-main.86/

[10] Yu, F. et al. *Beyond Pointwise Scores: Decomposed Criteria-Based Evaluation of LLM Responses*. arXiv:2509.16093, 2025. https://arxiv.org/abs/2509.16093

[11] Wang, P. et al. *Large Language Models are not Fair Evaluators*. arXiv:2305.17926, 2023. https://arxiv.org/abs/2305.17926

[12] Liu, N. F. et al. *Lost in the Middle: How Language Models Use Long Contexts*. arXiv:2307.03172, 2023. https://arxiv.org/abs/2307.03172

[13] Zhou, C. et al. *LIMA: Less Is More for Alignment*. NeurIPS 2023. https://arxiv.org/abs/2305.11206

[14] Dubois, Y. et al. *AlpacaFarm: A Simulation Framework for Methods that Learn from Human Feedback*. arXiv:2305.14387, 2023. https://arxiv.org/abs/2305.14387

[15] Singhal, P. et al. *A Long Way to Go: Investigating Length Correlations in RLHF*. arXiv:2310.03716, 2023. https://arxiv.org/abs/2310.03716

[16] Gao, L. et al. *Scaling Laws for Reward Model Overoptimization*. arXiv:2210.10760, 2022. https://arxiv.org/abs/2210.10760

[17] Park, S. et al. *Disentangling Length from Quality in Direct Preference Optimization*. arXiv:2403.19159, 2024. https://arxiv.org/abs/2403.19159

[18] Shao, Z. et al. *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models*. arXiv:2402.03300, 2024. https://arxiv.org/abs/2402.03300

[19] Sharma, M. et al. *Towards Understanding Sycophancy in Language Models*. arXiv:2310.13548, 2023. https://arxiv.org/abs/2310.13548

[20] Zheng, L. et al. *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*. NeurIPS 2023. https://arxiv.org/abs/2306.05685

[21] Panickssery, A. et al. *LLM Evaluators Recognize and Favor Their Own Generations*. NeurIPS 2024. https://arxiv.org/abs/2404.13076

[22] Min, S. et al. *FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation*. EMNLP 2023. https://arxiv.org/abs/2305.14251

[23] Mallen, A. et al. *When Not to Trust Language Models: Investigating Effectiveness of Parametric and Non-Parametric Memories*. ACL 2023. https://arxiv.org/abs/2212.10511

[24] Bowman, S. R. et al. *Measuring Progress on Scalable Oversight for Large Language Models*. arXiv:2211.03540, 2022. https://arxiv.org/abs/2211.03540

[25] Wang, X. et al. *Self-Consistency Improves Chain of Thought Reasoning in Language Models*. ICLR 2023. https://arxiv.org/abs/2203.11171
