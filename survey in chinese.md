任务型对话系统中的自然语言生成（NLG）模块，其输入是pipeline方法中上游模块DM模块的输出，即对话动作(dialogue act)，输出是自然语言回复。NLG的主要任务是将DM模块输出的对话动作（dialogue act）经一系列变换，转换成句法合法、语义准确的自然语言形式的回复。

## NLG传统方法

NLG的传统方法通常被分为四类：基于模板的方法(template-based)、基于句子规划的方法(plan-based)、基于类的方法(class-based)和基于短语的方法(phrase-based)。

### Template-based

基于模板的方法需要专家针对每个特定的领域设计对话模板，这些模板通常由两部分组成，一部分是固定不动的自然语言，另一部分则需要根据DM模块输出的对话动作进行填充。基于模板的方法较为简单，有了模板后，生成回复主要分为两步，第一步根据对话动作选择相应的模板，第二步对模板中缺失的地方进行填充。下面是一个简单的使用模板在餐厅预定领域生成回复的例子：

从DM模块接收到对话动作confirm(food=\$V)，意思是需要向用户确定要预定的是食物类型为$V的餐厅，\$V可能的取值是中国菜、意大利菜等，此对话动作对应的一个模板是“您是想要预定\$V餐厅吗？”。选定模板后，将food的值填到\$V的地方即可生成回复。

**优缺点：**该类方法的优点是十分简单、高效，产生的回复精准易控制且质量较高(依赖于模板集的质量)；缺点是可移植性差，每遇到一个新的领域都需要编写适用于相应领域的模板，人工编写和维护模板的工作量较大。

### Plan-based

基于句子规划的方法将NLG拆分为三个模块：句子规划生成(sentence plan generator)、句子规划排序(sentence plan ranker)、表层生成(surface  realizer)。

**句子规划生成模块**：任务是将输入的语义符号，如对话动作（dialogue act），映射到一个可以表示整条语句的中间形式，此中间形式即句子规划。句子规划的具体形式可以是句子规划树或者一些模板结构。

句子规划生成器通常由一系列语句合并的操作构成，这些合并操作可以将一系列低层的的谓词变元表示（最底层就是对话动作）组合成一个更高层的深层句法结构(DSyntS)。对于输入的一系列对话动作，结合的顺序通常是自底向上、自左向右。为了防止句子规划生成器生成比较差的句子规划树，有的工作引入一些修辞关系，通常是证明（justify）、对比（contrast）、细化（elaboration），在生成句子规划树之前，先对输入的对话动作进行排序。

句子规划通常以句子规划树的形式呈现。句子规划树是一个二叉树，叶节点是当前模块的输入，即一系列基础对话动作，内部节点是语句结合的操作（clause-combining operations）。每个结点也都有与之对应的深层句法结构。整条待生成语句的深层句法结构和根节点相对应。

**句子规划排序模块**：此模块的任务是对句子规划生成模块生成的一系列句子规划进行排序，并将得分最高的那个送给最后一个模块表层实现器进行自然语言实现。

此模块使用的算法通常是RankBoost算法，其使用m个指示函数（indicator functions） $h_m(x)$来表示一个句子规划，每个指示函数的取值是0或1，是使用一个特征加上一个阈值条件来计算的。将每一个指示函数乘上各自的系数再相加就得到这个样本的ranking score：
$$
F(x)=\sum_s \alpha_sh_s(x)
$$
系数$\alpha_s$就是待训练的参数。训练样本为一系列人工标注好的有序对(x, y)，其中x和y是同一个待生成句子的两个候选的句子规划，且x的优先级高于y。每两个候选者都存在一个这样的pair，故如果一个句子有20个候选，其训练样本就有20*19/2个有序对。

损失函数可以有多种选择，一个可能的损失函数如下所示，但相同的目标就是使$F(x)-F(y)$尽可能的大：
$$
Loss=\sum_{(x,y)\in T}e^{-(F(x)-F(y))}
$$
**表层实现器**：任务是将接受到的上一模块输出的句子规划转换成自然语言形式的最终回复。

转换过程是基于各个语句合并操作的规则，在句子规划树上，自底向上、逐层地将所有子节点合并成父节点，最终生成自然语言回复。

**优缺点：**
相比于之前的基于模板的方法，优点是应用了句法树，可以建模和生成复杂的语言结构。缺点是同样需要大量的领域知识，可移植性和可拓展性差，每到一个新领域都需要大量的人工工作，且效果并不比基于模板的方法好。

### Class-based

基于类的方法首先构建话语类和单词类的集合。此方法是统计方法，使用了语言模型，用来训练的语料库中的语句会被事先标注上语句类（utterance class）和单词类（word class）。基于类的方法分为内容规划和表层实现两个模块：在内容规划模块，使用一个双阶段的语言模型来决定回复中应该包含哪些单词类；在表层实现模块，使用语言模型来随机生成回复。

**内容规划模块：**任务是决定在生成的回复中应该包含哪些单词类，单词类即槽位。

此模块使用语料库构建一个两个阶段的语言模型。第一个阶段，模型根据给定的话语类别预测本句应该包含的单词类别的数量，模型可以表示为分布：$P(n_k|c_k)$，其中$c_k$是话语类别，$n_k$是词语类别的数量；第二个阶段，使用bi-gram语言模型，根据前一条语句中的单词类别集合来预测具体使用那些单词类别。在第二个阶段，由于单词类别的组合很多，训练bi-gram语言模型的过程会遭遇稀疏问题，通常的做法是做一些独立性假设，将单词类别集合到集合的bi-gram转换成单词类别到单词类别的bi-gram，简化问题。

**表层生成模块：**任务是随机生成自然语言回复。

训练阶段，一般会先对语料进行去词汇化(将单词替换成相应的单词类)，再使用去词汇化后的语料对每一个话语类别建立一个n-gram语言模型。针对某一个话语类别生成回复的时候，先使用相应类别的语言模型，用束搜索的策略生成多个回复，再结合内容规划模块的结果，设计一些启发式，来对语言模型生成的多个回复进行打分，并选择最高分的回复。最后用真实的值来填充其中的单词类，得到最终的自然语言回复。

**优缺点：**相比于之前的方法，基于类的方法由于使用了语言模型，该类方法在生成结果的流畅度、多样性上都有明显提高。缺点是对每个领域都需要手工创建类别集合，同样可移植性和可拓展性差，且语言模型的计算效率低，稀疏问题也未能得到很好的解决。

### Phrase-based

基于短语的方法是一种基于统计的数据驱动型方法，摆脱了手工制定模板的缺陷，将主要的人工努力从模板的设计和维护转移到数据标注上**。**训练前，首先要将训练集中的句子以短语为单位进行对齐标注；训练时，将标注的结果，即一系列无序的强制语义堆(mandatory semantic stack)送给语言生成模块，生成回复并对模型进行训练。基于对齐标注数据进行语言生成的方法有很多，比较通用的方法是使用动态贝叶斯网络（Dynamic Bayesian networks，DBN）生成自然语言回复。

**数据对齐标注：**任务是手工将语句分成短语序列，再通过映射自动得到语句的强制语义堆表示。

基于短语的方法需要对齐标注的数据。每一个短语会被自动映射到一个短语层次的语义堆（phrase-level semantic stacks），每一句话可以被表示成一个堆序列，每一个堆(stack)可以看作一个被压缩成一维的语义树(linearised semantic tree)。堆的最底层，即树的根节点是这句话的对话动作（dialogue act）；堆的其他层，即树的其他层级的节点有：实物的属性，如food，area，属性的值，如Chinese，特殊符号，如not。

堆会被分为两种：强制语义堆(mandatory stacks)是实体，记作$s_m$；可选的中间堆(optional intermediary stacks)不是实体，记作$s_i$，他们讨论的是对象的一般属性，或与对话动作相关联的、不在输入中的概念，如表示对话行为类型的短语，或子句聚合操作。

**语言生成模块**：任务是输入一个含有多个强制语义堆的集合$S_m$，生成最可能的回复序列$R^*=(r_1,r_2,...,r_L)$，其中$L≥|S_m|$。

定义该领域的可选的中间堆的全集为$S_i$，给定强制性语义堆的所有可能的堆序列的集合$Seq(S_m) \subseteq \{S=(s_1,...,s_L)s.t.s_t\in S_m \cup S_i\} $。语言生成模块使用了概率模型。假设标注所对应的语义树的两个叶子节点如果只有根节点这一个祖先节点，则这两个节点对应的短语是相互独立的，那么我们就可以使用贝叶斯公式对生成目标进行简化。首先，在所有可能的堆序列$Seq(S_m)$中通过最大化$P(S|S_m)$找出最佳的语义堆序列$S^*$，其中$S^*$包含所有$S_m$和某些$S_i$的组合。然后，在所有短语序列上通过最大化$P(R|S^*)$即可找到最佳语义堆序列下的最可能的自然语言回复$R^*$。即
$$
R^*=\mathop{argmax} \limits_{R=(r_1,...,r_l,...)}P(R|S^*)
$$

$$
with\ S^* =\mathop{argmax} \limits_{S\in Seq(S_m)}P(S|S_m)
$$

求解这两个方程的经典方法是利用动态贝叶斯网络进行训练，使用任意的推断算法，如连接树推断算法（the junction tree inference algorithm）进行$S^*$和$R^*$的查找。

**优缺点：**基于短语的方法是数据驱动的方法，相比于其他可训练语言生成模型，其主要优点是不再需要手工写成的生成器（generator）。此外，因为训练集的堆本质上是一个树结构，基于短语的方法可以用较少的参数来建模长范围的依赖关系和特定领域的惯用短语。缺点在于需要很多的语义对齐处理，同样难以移植和拓展，标注数据的方法对结果影响很大，需要尝试标注方式。