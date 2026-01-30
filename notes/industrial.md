主流 AI Agent 系统的记忆机制调研

AI Agent 记忆类型概述

AI Agent（自主智能体）通常需要两类记忆机制 ￼ ￼：
	•	短期记忆：指当前对话或任务的上下文，在一次会话/推理过程中存储最近的交互信息。通常由LLM的上下文窗口承载（例如GPT-4有8K-32K tokens上下文）。短期记忆让Agent能“记住”当前会话中的用户提问和自身回答 ￼。由于上下文窗口有限，Agent常采用消息截断或摘要等策略管理长对话 ￼。
	•	长期记忆：跨会话或长时间跨度保留的信息，例如用户偏好、先前完成的子任务结果等 ￼。这通常需要将信息存储到外部数据库或文件中，在需要时检索增强（Retrieval-Augmented Generation, RAG）将相关内容提取并注入LLM上下文 ￼。长期记忆机制使Agent能够在多轮对话乃至多次独立会话中持久地记住关键信息 ￼。

在实现上，常见的存储结构包括向量数据库（将文本转为嵌入向量以便相似检索） ￼、键值存储/JSON（结构化保存事实或配置） ￼，以及简单的文件或内存缓存等 ￼。此外，为避免短期上下文爆炸，很多系统支持自动摘要/压缩旧对话：例如将早期对话总结为简短概要以释放上下文空间 ￼。下文将分别介绍开源框架和主流闭源产品中的记忆机制及其存储策略，并说明它们是否采用上下文图谱/知识图谱来建模记忆关系，以及是否支持基于图结构的推理与检索。

开源自主Agent的记忆机制

LangChain 框架的记忆模块

LangChain作为流行的LLM应用框架，提供了丰富的记忆模块支持 ￼。在新版LangChain中（引入LangGraph架构），记忆被划分为短期和长期两部分：
	•	短期记忆（线程级）：LangChain通过会话消息缓冲来跟踪当前对话历史 ￼。这些历史消息保存在Agent的状态中，并利用检查点机制持久化，可随时恢复会话 ￼。短期记忆可以包含对话消息列表，以及运行中产生的其他状态（如工具检索到的文档或用户上传的文件） ￼。为管理长对话，LangChain提供了多种策略，例如窗口内存（只保留最近N条消息）、消息过滤/删除以及自动摘要旧消息 ￼。例如，开发者可让Agent定期将较早的对话内容总结为概括性语句并替换详细消息，从而既保留上下文要点又不超出模型上下文长度 ￼。
	•	长期记忆（跨会话）：LangChain允许Agent将信息存入持久存储以跨会话复用 ￼。具体实现上，LangChain的LangGraph引入了Store接口，将长期记忆以JSON文档形式保存在数据库或向量库中 ￼。每条记忆归属一个namespace（如用户ID或应用场景）以及唯一key，支持层级组织和跨命名空间检索 ￼。典型用法是将某用户的偏好或资料存入向量数据库，并在新对话时通过语义查询检索相关记忆 ￼。例如，LangChain提供Profile和Collection两种管理方式：前者将用户信息维护为单个JSON档案并在每次对话后更新 ￼ ￼；后者则将不同片段作为独立文档集合存储，需要在检索时聚合结果并处理冲突更新 ￼。开发者可根据应用需要选择方案。长期记忆的读写通常通过工具（Tool）接口进行：LangChain允许定义自定义工具让Agent调用，从Store中查询或写入信息 ￼ ￼。例如，可以有“查询用户资料”的工具在被Agent调用时返回Store里该用户的存档数据 ￼。这种设计确保了LLM不会自动将大量长期记忆塞入每次Prompt，而是按需检索，控制上下文长度。

LangChain当前主要借助嵌入向量+过滤来检索记忆，并不直接使用知识图谱存储关系。在LangGraph存储中，记忆JSON可以有人为定义的键值结构，但系统并未显式构建实体-关系图谱。目前记忆检索以语义相似为主 ￼（可对内容或元数据过滤后按嵌入相似度排序返回）。不过，LangChain的设计已考虑不同记忆类型对应不同用途，例如语义记忆（事实）、情景记忆（事件经过）和过程记忆（规则）等 ￼ ￼。研究也有将这些类比于人类的语义/情景/程序记忆体系 ￼。总的来说，LangChain提供了完整的记忆框架：短期会话记忆管理、长期向量库持久化，以及一些内置工具（如ConversationBufferMemory、ConversationSummaryMemory、VectorStoreRetrieverMemory等）来方便地集成各种记忆策略。

Auto-GPT 的记忆机制

Auto-GPT是2023年兴起的自主代理应用，它在架构上明确区分了短期和长期记忆 ￼：
	•	短期记忆：Auto-GPT利用LLM自身的对话上下文作为短期记忆，上限由模型token窗口决定（GPT-3.5约4千字，GPT-4可达8千或更多）。在每个思考循环中，Auto-GPT会将最近的对话内容和Agent思考“Chain-of-Thought”放入Prompt，以便模型结合最新上下文进行推理 ￼。Auto-GPT也面临上下文窗口限制的问题，开发者反馈其长任务容易遗忘先前细节或陷入循环 ￼。为此，一些改进版本引入了消息摘要或检查点：例如，当任务非常长时，Auto-GPT可能将阶段性成果记录下来，并清理或总结中间过程，以减轻Prompt长度。不过，原版Auto-GPT主要依赖长期记忆来弥补短期上下文的不足 ￼。
	•	长期记忆：Auto-GPT的突破在于引入了持久向量数据库来存储代理的经验和信息 ￼。官方实现允许配置多种向量存储后端，如LocalCache（本地JSON文件存嵌入）、Redis、Pinecone、Milvus或Weaviate等 ￼。运行时，Agent会将有用的信息（网页内容摘要、推理得到的结论、重要提示等）通过OpenAI的embedding接口转换为向量并存入内存库 ￼。当Agent需要回忆时，会对向量库执行语义查询，找出与当前任务相关的记忆片段，将其附加到Prompt前文，以辅助决策 ￼ ￼。例如，有用户在Auto-GPT中配置Pinecone实现长期记忆，用于存储中间结果、笔记，Agent在后续步骤能通过相似搜索找到此前整理的要点 ￼。

Auto-GPT的长期记忆类似于检索增强生成(RAG)流程 ￼：将外部知识库当作延展记忆。事实上，Auto-GPT的文档称其长期记忆“通常使用RAG或向量数据库”，以便“让Agent能随时间推移记住相关细节、处理更大的知识库” ￼。值得注意的是，Auto-GPT默认会将关键长久内容固定在Prompt开头（作为系统提示）来避免被模型遗忘 ￼。有Issue指出Auto-GPT会在每次对话开头加入上一轮决定保存的长期记忆摘要，从而“钉住”重要信息不被上下文裁剪掉 ￼。开发者也希望Agent能感知自身的上下文用量，及时将信息转存长期记忆以防溢出 ￼。

存储结构方面，Auto-GPT通过.env配置选择Memory后端 ￼。默认Docker部署用Redis内存库（键值数据库），本地运行默认LocalCache（JSON文件） ￼。如果使用向量数据库如Pinecone或Weaviate，则需要提供API密钥，Auto-GPT会将向量索引存储在云服务上 ￼ ￼。Auto-GPT也可以将内容写入普通文本文件（例如Agent会把长网页保存为文本文件，并在需要时调用“read_file”工具读取），这种磁盘I/O也是简单的长期记忆形式 ￼。

记忆管理策略：Auto-GPT在记忆方面还比较朴素，主要由Agent逻辑决定何时“学习”或“回忆”。例如，它有时会在产生新的重要信息时调用保存命令，将其载入长期向量记忆；在每轮思考开头，会根据当前任务从长期记忆中检索可能相关的信息添加到提示 ￼。有些社区优化版本为Auto-GPT增加了总结记忆功能，定期将对话日志浓缩成要点存储，以提升检索效率和减少冗余 ￼。一篇技术博客指出：“AutoGPT和CrewAI使用向量存储结合摘要记忆，周期性地将较旧的交互压缩成高层次要点” ￼。这种做法使Agent不会遗忘总体进展：即使详细对话不在短期窗口中，Agent仍可从摘要中获取过去的关键决策和结论。不过，这并非Auto-GPT官方内置功能，而是社区实践。Auto-GPT本身并未使用知识图谱来表达记忆关系——它存储的信息基本是独立的文本块嵌入，不保留显式的关联结构。因此，它无法直接“理解”两个记忆之间的人物关系等联系，除非这些联系以文本形式被模型读入并推理。

（注）其他开源代理

除了LangChain和Auto-GPT，诸如BabyAGI、AgentGPT等项目也探索了Agent记忆。BabyAGI采用一个任务队列反复执行的简单循环，其“记忆”主要是任务列表和向量数据库（用于存储处理过的信息供后续任务查询） ￼。相较Auto-GPT，BabyAGI更轻量化，记忆管理也更依赖向量检索和人工设定的任务总结。另有一些框架如CrewAI提供了更复杂的记忆子系统（支持短期、长期、实体记忆等）和事件机制来监控记忆读写 ￼。总体而言，大多数开源自主Agent都采用相似的记忆范式：利用LLM上下文作为短暂记忆，同时借助外部向量存储/数据库保存长久信息，并通过一定策略（摘要、截断、向量检索）在二者之间平衡 ￼。不同框架在易用性和扩展细节上有所差异，但在是否引入知识图谱作为记忆表示方面，当前都还处于探索阶段，并非主流做法。

闭源AI产品的记忆机制

OpenAI ChatGPT（及Code Interpreter）

ChatGPT作为OpenAI的对话模型，最初仅具备短期会话记忆，即在同一对话线程中记住先前的对话内容。它并没有真正的长期记忆：每个新对话对模型而言是隔离的。 ￼ ￼（OpenAI官方社区明确表示：“ChatGPT不会跨会话记忆，上一次对话内容不会被下一次对话自动调用” ￼）。ChatGPT短期记忆的容量受模型上下文长度限制：GPT-3.5约4096 tokens，GPT-4增强到8192或32768 tokens。这意味着当对话超过这个长度时，最早的消息将被模型遗忘（在实现上通常是接口截断早期消息）。ChatGPT不会自动总结超长对话，导致用户在长对话中可能感到模型“逐渐忘事”。

转折发生在2023-2025年间：OpenAI开始尝试为ChatGPT引入跨会话记忆功能，以提升个性化和上下文衔接。2025年4月，有报道指出ChatGPT推出了可选的长程记忆功能，允许模型“利用用户所有过去的对话来提供更加个性化、有上下文的回答” ￼。该功能首先向Plus/Enterprise用户开放（欧洲地区因GDPR暂不可用） ￼ ￼。开启后，ChatGPT能够访问用户的完整历史聊天记录，在回答时检索相关的过去内容 ￼。这标志着ChatGPT从一次性工具向“持续助手”转变 ￼。比如，在专业场景下，有长程记忆的ChatGPT可以记住用户反复提及的战略问题、先前项目细节，甚至保持用户偏好的回应风格 ￼。为保护隐私，OpenAI提供了内置的记忆管理接口：用户可以查看、编辑或删除存储的历史“记忆”，也可以完全禁用该功能 ￼。

存储与机制：ChatGPT的长期记忆实现细节未完全公开。据推测，OpenAI为每个用户维护了一个向量化的对话数据库或类似存储，用于个性化检索。这类似于RAG，将用户历史作为知识库供模型查询 ￼。例如，当用户再次讨论某主题，系统可提取他过去对此的陈述作为提示的一部分。不过，与直接的向量检索不同，OpenAI可能在服务端进行更复杂的过滤和调度，确保调用历史不会引入过多噪音或隐私风险。2023年OpenAI还推出了自定义指令功能，让用户保存一些偏好说明（如“我是一名教师，说话口吻正式”）供所有对话默认使用。这实质上是另一种形式的长期记忆，只是存储的是用户元信息而非具体对话。综合来看，ChatGPT直到2025年才正式具备跨会话记忆能力，此前主要依赖短期上下文。当前它的长程记忆相当于全量历史的向量搜索 + 人工选择 ￼（官方强调用户有控制权），并未报告使用知识图谱等结构化关系存储。

Code Interpreter/高级数据分析作为ChatGPT的特殊模式，引入了工作空间记忆的概念。在这个模式下，ChatGPT可以在沙盒中执行Python代码并读写文件。文件系统就充当了扩展的记忆：模型可以把较大的中间结果保存成文件，再在后续对话通过“阅读文件”来获取，不必把所有数据放进Prompt。例如，用户上传一个大文件，模型先读取并生成摘要，接着可以根据需要多次访问文件内容，而无需用户每问一次就重新提供数据。这种环境状态是会话内有效的临时记忆。变量值、生成的图表、保存的CSV等都存在于会话的后端容器里，直到会话结束或环境重置。虽然Code Interpreter不提供跨不同聊天会话的记忆，但在单次会话中，它能存储更多信息（通过文件/变量）而不占用LLM上下文。例如，它可以在内存中缓存一个大型数据集的处理结果，用户后续提问时模型只需引用结果摘要，而不用再次处理整个数据。这提高了长对话的效率和一致性。可以认为，Code Interpreter模式下，ChatGPT有短期对话记忆+持久会话状态记忆两种：前者是传统上下文窗口，后者是代码工作区。两者结合，使得在分析数据等复杂任务中，Agent不会因为token限制忘记前面算过的结果。需要注意的是，这种文件/环境记忆仅限于当前对话，不会长期保存（除非用户手动下载结果）。

总之，ChatGPT目前的记忆机制特点：短期依赖大窗口LLM，自带基本的截断策略；长期刚刚起步，通过检索历史对话和用户指令，实现有限的个性化连续对话 ￼。它未见公开采用知识图谱结构，大多仍是基于文本Embedding的语义存取。OpenAI更强调通过增加上下文长度和引入检索来扩展记忆，而非让模型内部建立显式的关系网络。

Anthropic Claude

Claude（Anthropic研发）以超长上下文能力闻名。Claude 2模型在2023年推出时支持100K tokens的上下文窗口 ￼——相当于约75,000字的文本。这意味着Claude在单次对话中可以“记住”一本书或数百页文档的内容 ￼。因此Claude主要通过扩展短期记忆缓解了很多场景下对长期记忆的需求：用户可以在一次对话里不断追加资料，而Claude不会像普通模型那样很快忘记开头的信息。这使Claude特别适合处理长文档问答、复杂代码库分析等任务 ￼。有用户报告Claude能在长对话中保持上下文一致，总结出贯穿对话的用户偏好和性格特征 ￼。

然而，超长上下文并不等同于真正的长期记忆。Claude在不同会话之间仍然不能自动共享记忆（除非通过用户提供的系统提示）。Anthropic也意识到仅靠大窗口不足以应对跨会话的连续性，因此开始开发持久知识库功能。据报道，Claude计划引入“多知识库（project library, workflow library等）”让用户管理不同类别的长期记忆 ￼。这类似于为Claude增加项目级、工作流程级的上下文存储，用户可以预先将资料上传到这些知识库。当对话需要相关信息时，Claude可从对应库中检索 ￼。实际上，Anthropic在Claude的企业版中已经提供Claude知识基地 (Claude Knowledge Bases)，允许将文档集合与聊天机器人关联，Claude会对这些文档执行检索以回答问题 ￼。这正是典型的RAG架构：Claude通过内置的“知识搜索工具”从用户上传的文档中提取相关段落，结合对话上下文回答 ￼。

Anthropic官方将这种能力称为**“进阶记忆”或持续知识**。有分析文章称：“Anthropic在为Claude开发持久‘知识库’以实现长期任务连续性和项目级记忆” ￼。工程上，Claude的实现大概是维护一个外部索引（例如向量数据库）存储每个“项目”下的文档向量。当对话属于某项目时，Claude可以用一个专门的tool调用搜索接口，检索出与用户提问相关的段落，并将其填入Prompt ￼。这种记忆不在模型内部，而是在Claude CoPilot/Claude API层完成。Anthropic还在尝试更结构化的记忆表示。社区讨论提到Anthropic考虑过**“progress files”**等设计，把记忆存成可查询的知识库，让模型可以进行推理 ￼。这暗示可能引入一定的结构（例如时间戳版本的事实记录）。

目前Claude的记忆管理主要靠其大容量上下文来实现“短期不忘”。对于特别长的单次对话，Claude也有策略：当消息接近token上限时，会自动舍弃最早的工具输出或交互内容，以腾出空间 ￼（Anthropic文档提到Claude在对话窗口逼近极限时，会移除旧的工具调用结果和交互）。这属于简单的截断策略，不涉及摘要。由于100k窗口非常大，Claude较少需要摘要优化。但当需要跨对话连续时，Anthropic更倾向于引导用户使用知识库功能，即显式地将信息存入长期库，由模型检索而非让模型“记住”在参数中。

在知识图谱/关系记忆方面，Claude目前并未公开支持内部的图谱推理。Anthropic更加强调伦理约束和安全原则（Claude采用“宪法AI”方法），这属于模型行为规则记忆的一种（可以看作程序性记忆的一部分）。ActuIA的分析指出：“Claude提供一种RAG式的记忆，将对话记录和外部知识库结合，并且非常强调伦理和对齐” ￼。其中并未提及Claude使用知识图谱。不过，Anthropic的长期愿景包括让AI具备持续学习能力 ￼。CEO Dario Amodei曾表示“持续学习最终不会像看上去那么难” ￼，这或许意味着未来Claude会增加自动积累知识的机制。目前，可以将Claude定位为通过超长上下文+插件式检索实现记忆的系统，暂未发现Graph数据库或知识图谱直接融入其记忆模块。

Inflection AI 的 Pi

Inflection AI的 Pi 定位为“个人AI伴侣（Personal Intelligence）”，因此非常强调长久的个性化记忆。与ChatGPT等通用助手不同，Pi的设计宗旨是与用户建立持续关系，充当贴心朋友 ￼。这体现在Pi会“记住”用户过去说过的话、经历的情绪状态，并在后续对话中体现出连续性。

跨会话记忆：Pi支持跨设备、跨平台的对话连续 ￼。无论用户是在手机、网页还是短信上与Pi聊天，Pi都会同步更新同一份对话历史和用户偏好 ￼。官方评测提到：“Pi能够维护详细的对话记忆、用户喜好和关系上下文，并在所有平台上同步，确保不论何种访问方式，Pi都记得关键细节” ￼。这表明Inflection在后台为每个用户存有一个统一的用户模型或资料库。其中不仅包括对话内容，还有推断出的用户偏好、人格特征等。比如，如果用户在上一次聊天提到自己喜欢某个运动，Pi下次交谈时可能会主动问及相关话题，表现出对用户生活的了解。

实现：虽然Inflection未公开技术细节，但可以推测Pi使用了数据库+嵌入检索结合的方案。它可能将每个用户的历史对话分段向量化，用于个性化的语义检索。当新对话到来时，Pi会调取用户过往相关的谈话摘要，融入回复生成。工具Stack的测评也指出：“Pi能够记住过去的对话，会追问后续问题，对用户生活表现出好奇，在多次交互中保持上下文” ￼。这些都需要对历史进行一定程度的结构化。Pi或许会为每个用户维护一个个人档案（Profile），包含用户提供的关键信息（名字、兴趣、关系等）。事实上，LangChain等框架也支持类似Profile的记忆形式 ￼。Pi很可能有更复杂的管道，例如情感记忆：记录用户在某些话题上的情绪，从而未来回应时给予特别的关怀 ￼。

短期记忆方面，Pi的LLM模型Inflection-2也有一定上下文长度，但据用户反馈，Pi在长对话中有时会重复和遗忘，表现出长对话记忆有限的迹象 ￼。这说明Pi内部可能对对话长度仍有限制（也许数千token级别），超出后可能采用总结或话题分段策略。工具Stack评论的缺点之一提到：“在扩展对话中，Pi可能反复回到相似话题或重复措辞，暗示其长期记忆和创造力的局限” ￼。这提示Pi或许没有复杂的摘要算法，而是依赖模型自身对长对话的承载能力。当对话过长时，模型可能丢失上下文细节，只能围绕最近话题打转。

总的来说，Pi作为私人AI，长期记忆是其卖点之一。ActuIA文章也提到：“像Inflection的Pi这类项目专注于情感记忆，旨在与用户建立持续互动关系” ￼。Pi不一定使用了显式知识图谱，但它注重关系和偏好的记忆，这本身就是一种“关系上下文”。例如，Pi可能记得用户和某亲友的关系，在对话中做出符合该关系的回答。这种关联可能由模型从上下文推理得到（隐式）或由系统存储用户提供的信息（显式）。目前没有公开证据表明Pi内部构建了知识图谱进行推理，但关系上下文的保存无疑是其功能点。

Google Gemini

Gemini是Google DeepMind在2024-2025年研发的下一代多模态大模型。作为Google生态的一部分，Gemini非常强调与用户个人数据和应用环境的整合，被称为提供“Personal Intelligence（个人智能）”的新特性 ￼。Gemini的记忆机制可以从几个层面来看：
	•	超大短期上下文：据Google官方介绍，Gemini第三代模型拥有100万token上下文窗口 ￼。这个规模远超其他模型（是Claude 100k的10倍以上），意味着Gemini在一次会话中可以处理海量信息。这奠定了强大的短期记忆基础，使其能够一次性读取用户整个邮件收件箱或照片库的描述等 ￼。然而，Google也指出，仅靠1M tokens仍不够涵盖用户所有数据，因为“单是邮件和照片的累计上下文就经常数倍于此” ￼。因此，Gemini引入了一个动态的**“上下文打包 (context packing)”机制：实时筛选出用户数据中与当前查询最相关的部分，综合成模型的工作记忆输入 ￼。简单来说，Gemini不会盲目把所有历史都塞进那100万tokens里，而是从中挑选恰当的信息片段填充。这实际上是一种智能摘要/检索**：对候选信息打分，选出有用的内容送入LLM上下文，相当于自动的RAG。 ￼
	•	工具和检索融合：Gemini的Personal Intelligence引擎还强调工具调用和密集检索的结合 ￼ ￼。当用户向Gemini提问涉及个人信息（如“我下周的航班几点”），Gemini会触发专门的工具去搜索用户的Gmail/日历等找到航班预订邮件；又如用户问“给我推荐酒店，基于我过去去过的地方”，Gemini会检索用户Google Maps访问记录或之前的邮件评价 ￼ ￼。这一切由背后的Personal Intelligence Engine协调 ￼。这引擎会安全地连接用户在各Google产品中的数据仓库，在需要时执行类似数据库查询或搜索，然后把结果反馈给Gemini模型 ￼。Gemini模型本身也具备更强的推理和工具使用能力，能够理解复杂的个人上下文（如搞清用户家庭成员之间的关系）并知道何时调用哪个工具获取补充信息 ￼ ￼。例如，Gemini可以把用户在Gmail里的行程邮件、在Photos里的风景照EXIF信息、在YouTube上的历史都调出来综合考虑，回答一个复杂问题。
	•	长期记忆和个性化：Gemini致力于成为真正的个人助手，因此会持续地积累用户画像。其Personal Context功能会“主动从过去的所有聊天中构建用户兴趣的长期档案，以影响未来回答” ￼。这表示Gemini不仅对话时用RAG，即使用户不提问，它也在后台分析用户的历史对话、搜索记录等，更新对用户的了解。这些被保存在用户的Personal Memory中 ￼。Google提供界面让用户管理这些个性化数据（可开关“Personal Context”并选择哪些来源接入） ￼。在企业版Gemini中，也有个性化和记忆配置选项，说明它确实维护了一套针对每用户的记忆存储 ￼。Gemini的长期记忆可看作是对用户各类数据的索引+理解：索引部分通过工具可以检索到原始数据，而理解部分则让模型知道这些数据中的关系和意义（比如照片中的人是用户的表亲等）。这接近于知识图谱的概念——Google本就拥有强大的知识图谱技术，用于理解公共知识。而Gemini则在尝试构建个人知识图谱：把用户的邮件、联系人、日历事件、照片元数据等链接起来，供模型查询推理。所以ActuIA文章提到：“Gemini在Google Workspace生态中融合跨上下文元素，预示一种分布式记忆形式，以文档为中心” ￼。这一方面指Gemini深度集成Google文档/邮箱等应用（文档型数据），另一方面暗示Gemini的记忆是“分布式”的——散布在各种数据源，通过引擎按需提取，而非一个静态大脑。

综合而言，Google Gemini代表了混合记忆系统的前沿：超大模型上下文提供即时记忆容量，加上多源信息检索实现事实存储，最后借助关系推理整合信息为用户提供个性化帮助 ￼ ￼。在是否使用图谱方面，虽然Google未明确声明“知识图谱”二字，但Gemini能理解人与人之间的关系、事件时间先后等，就离不开对信息的结构化表示 ￼。可以推测Personal Intelligence Engine内部将用户数据建模为带节点关系的索引（某种私有的知识图谱或数据库），Gemini通过查询这些结构化关系获得答案。例如，它知道某封邮件是用户叔叔发的邀请函，那么“叔叔-邀请-你”这层关系需被显式识别出来才行。总之，Gemini把检索、图谱和大模型融为一体，被视为迈向“真正个人化AI”的一步 ￼。

其它：Meta等的动向

值得一提的是，其他大厂也在探索Agent记忆的新路径。Meta据报道在开发社交助手，强调关系型记忆，会整合用户在社交平台上的关系网络和情感偏好，以提供更贴心的连续对话体验 ￼。这几乎肯定涉及到社交图谱（用户好友、关注内容等）的运用。另如Character.AI等对话产品，也以长期记忆和个性化见长：它会根据用户与虚拟角色过去的互动，调整角色回应（有点类似剧本记忆）。总体趋势是：长程记忆将成为下一代AI助手的标配，只是各家的侧重点不同——有的侧重工作效率（如ChatGPT企业版整合业务知识），有的侧重情感陪伴（如Pi、Character.AI），有的依托生态系统（如Gemini深挖Google数据）。然而，实现长期记忆的挑战在于取舍：如何让AI知道什么该记住、什么应忘记，以既满足个性化又避免陈旧和偏见 ￼。正如一篇评论所言：“真正的考验也许不在于记忆多少，而在于是否能选择遗忘” ￼。

基于图谱的记忆与关系推理

针对“Agent是否使用上下文图谱/知识图谱建模记忆”这一问题，我们需要区分当前主流实践和前沿探索：

主流产品中的情况：目前列举的LangChain、Auto-GPT、ChatGPT、Claude、Pi、Gemini等，大多数并未公开使用显式的知识图谱来存储记忆。它们的记忆更多是向量语义内存或文档检索形式：即把过往内容当作独立片段存储，靠语义相似度召回 ￼。这种方式简单有效，但缺点是丢失了片段之间的关系 ￼。例如，向量内存能找出“咖啡”相关的过去对话，但不知道这些对话里隐藏的联系——也许用户每次提到咖啡都提及同一家店、每周二去买 ￼。如果没有关系链，Agent难以推理出更深的模式（如用户咖啡偏好随时间变化）。

图谱记忆的优势：知识/上下文图谱指将记忆表示为节点+边结构，每个信息项作为节点，节点之间通过关系相连 ￼。这样Agent不仅知道孤立事实，还能沿着关系链追溯因果、归类聚合。例如，图谱记忆可以表示：“用户喜欢的咖啡 = 拿铁；拿铁来自Starbucks；上次购买=周二”之类的关联。当用户再聊咖啡，Agent可通过图谱快速定位相关节点（咖啡->拿铁->Starbucks）并了解上下文（周二买的） ￼。相比之下，向量搜索只能模糊地找到过去提到“咖啡”的句子，无法直接获知这些句子背后的结构 ￼。正如一篇对比总结：“向量记忆检索相似对话，但各段彼此独立；图谱记忆则保留信息随时间如何连接，让AI能基于关系推理、追踪偏好演变，并结构化地回忆” ￼。

新兴的图谱记忆方案：近年出现了一些专门增强Agent记忆的开源解决方案。例如：
	•	Zep：一个为LLM应用提供持久记忆的服务。最新版本的Zep引入**时间知识图谱（temporal knowledge graph）**来跟踪事实随时间的演变及关系变化 ￼。这意味着Zep不仅存储对话内容，还记录每条知识的时间戳和版本。如果用户半年后改变了偏好，旧知识会被标记过期，以免Agent检索到过时信息 ￼。Zep的图结构让Agent可以查询“某人管理哪个办公室”这样的关系型问题，并随着事实更新自动调整图谱节点 ￼。当然，引入图谱也带来复杂性，Zep团队指出这种时间图增加了系统复杂度，并非所有简单Agent应用都需要 ￼。
	•	LangChain LangGraph Memory (LangMem)：前文介绍的LangChain已经包含一些图谱思维的元素。LangChain的长期存储支持层级命名空间（User->Session等）组织记忆 ￼。而社区也有在LangChain上层封装的LangMem方案，提供更显式的关系建模能力。例如LangMem允许开发者将特定事实以三元组形式存入LangGraph Store，并通过自带的Memory工具维护这些关系 ￼ ￼。不过这种用法需要开发者自己设计知识表示，LangChain本身只是提供了底层API ￼ ￼。LangMem被指出的限制之一正是与LangGraph框架深度绑定，以及记忆操作需要显式触发，没有自动抽取 ￼。
	•	Mem0/MemU：这是近期的一个开源记忆中间件，主打图增强的记忆。Mem0提出一个“三层记忆架构”，自动从对话中抽取事实、生成摘要，组织成分层的用户->会话->Agent多级记忆，并结合向量+图混合检索 ￼ ￼。它的思想是用向量检索快速锁定相关主题，再用图谱精细追踪其中的关系细节，从而实现检索速度和准确度的兼顾 ￼ ￼。例如Mem0声称将向量检索与图检索结合，使响应速度提高91%、token消耗降低90% ￼。这印证了图谱在节省上下文长度上的潜力——因为关系明确，Agent不需要反复读取无关文本。在Claude宣布开发持久内存的消息后，Mem0团队甚至宣称“Anthropic终于认识到AI需要真正的记忆系统” ￼。可见业界对图谱记忆的重视。
	•	其他：还有一些框架如Letta（提供Agent自行编辑内存块的运行时，带REST API）、Supermemory（个人AI助理的记忆库，将笔记、文档等存入“个人记忆库”）等 ￼ ￼。这些探索各有侧重：Letta允许Agent自主决定哪些记忆留在上下文，哪些存档 ￼；Supermemory则偏向个人用户整合各种数据作为记忆 ￼。它们共同点是在尝试突破“仅向量、不关系”的瓶颈，让记忆更可控、更结构化。

支持基于图结构的推理/检索：一旦记忆存储为图结构，Agent就有机会进行基于图的推理，比如路径查找、模式发现等。然而，目前多数主流LLM本身并不能直接查询图数据库，需要借助Tool或后端服务。因此，上述框架通常提供API接口供Agent调用。一些研究者也尝试让LLM输出图查询语言（如Cypher）再执行，以检索知识图谱内容，然后返回给LLM。实际产品中，这种深度图推理尚未大规模应用，更多是用于企业内部知识管理等垂直领域（例如在安全可控的企业知识图上回答复杂业务问题）。

总的来看，上下文图谱作为记忆的“关系层”，在2025年前后开始兴起并被认为是下一步的重要方向 ￼。但对于ChatGPT、Claude这类现有系统，短期内仍主要依赖大模型自身的参数记忆和向量语义检索来实现上下文记忆功能。我们可以预见，随着需求增长，未来的Agent会逐步引入更多图谱元素：把用户的长期交互转化为知识图谱（人物、事件、偏好之间的网络），再配合LLM强大的语言推理，实现真正深度定制和举一反三的能力 ￼ ￼。

下面的表格对本文提及的主要系统的记忆机制进行总结比较：

系统	短期记忆机制	长期记忆机制	存储与检索结构	记忆压缩/总结	是否图谱化记忆
LangChain (开源框架)	会话消息缓冲记录当前对话，提供窗口裁剪、过滤和摘要等管理长对话的策略 ￼。短期记忆作为Agent状态，可持久化checkpoint方便恢复 ￼。	提供LangGraph持久存储，将记忆以JSON文档存入命名空间下，可跨会话按需取用 ￼。支持语义向量检索，Agent通过Memory工具调用读写长期记忆 ￼。	默认使用向量数据库（或内置InMemoryStore）存储嵌入向量及原始内容 ￼。采用<用户ID,上下文>命名空间组织层级记忆 ￼。检索可按元数据过滤并按向量相似度排序返回 ￼。	支持多种策略：可截断旧消息或将旧对话自动总结后替换，以节省token ￼。也支持将重要信息提取后写入长期存储（由开发者决定调用时机）。	部分支持：记忆以结构化JSON保存，可视为简易知识库，但未显式建立实体关系图谱。需开发者自行设计关系表示或使用LangGraph扩展。默认记忆检索基于文本Embedding，无内置图谱推理。
Auto-GPT (开源代理)	利用LLM上下文作为短期记忆，循环Prompt中包含最近对话和思考。受限于模型token窗口（GPT-4等） ￼。无自动摘要机制，长对话靠向长期记忆转存或牺牲早期内容 ￼。	使用向量数据库实现持久记忆 ￼。重要信息通过嵌入存储到如Pinecone/Weaviate/Redis等后端 ￼。每步决策前对向量库执行相似查询，将相关记忆插入Prompt上下文 ￼。可配置在多次运行间保留Redis内容，实现跨Session记忆 ￼。	JSON文件、本地或云向量数据库等皆可用 ￼。默认LocalCache(文件)或Redis，支持 Pinecone/Milvus等云向量库 ￼。通过embedding将文本表示为向量存储，检索时embedding查询找相近向量内容 ￼。此外Agent也会把信息保存到磁盘文件，供后续读取，作为另一种形式的外部记忆。	部分支持：原版主要靠向量检索，不自动摘要对话。但社区实践中，引入了定期总结日志功能，将过往步骤浓缩为要点存储，供后续参考 ￼。这降低了长链路任务的遗忘率。短期内存超限时，Auto-GPT倾向于发出警示或尝试将内容保存再清除Prompt。	否：Auto-GPT未使用知识图谱建模记忆。所有记忆片段独立存储为文本或向量，没有显式关系链接。Agent无法直接基于记忆间关系推理（除非关系本身写在记忆文本中）。
ChatGPT (闭源，OpenAI)	每个对话线程维护消息历史作为上下文，窗口上限取决于模型（GPT-4最大32K tokens）。超过长度时旧内容被截断舍弃。ChatGPT会尝试利用最近若干对话回答，之前的超出上下文就遗忘。无内置长对话自动摘要（用户也许需手动要求总结）。	2025年起支持可选的跨会话记忆（Chat History） ￼。开启后，模型可访问用户过往所有对话，用于个性化回答。例如能引用之前聊天提过的事实。 ￼。用户可管理这些长期记忆（查看/删除） ￼。实现上 likely 将历史对话嵌入索引，实时检索相关片段作为隐式系统提示。除此之外还有“自定义指令”功能，用于跨会话保存用户偏好说明。	短期存于对话缓存在服务端，随请求发送给模型。长期记忆存储在OpenAI云端数据库（每用户的会话索引）。具体结构未公开，可能使用向量索引+元数据过滤。检索逻辑在服务端，模型并不知道“记忆”存在形式，只看到结果融入的提示。	可能有：未公开细节。据用户体验，ChatGPT长对话可能偶尔总结内部状态以保持重要信息（也可能没有）。新推出的长期记忆功能本质上就是自动检索过去内容，相当于一种高级的摘要调用（从所有历史中挑选相关对话片段）。OpenAI强调用户在长程记忆中的控制权，暗示系统不会不加区分地引入所有历史，只会选取精炼要点。	否：暂无迹象OpenAI使用知识图谱来存用户记忆。ChatGPT的长期记忆基于过去对话文本，不推理人物关系或事件网络（除非模型读入这些文本自行推理）。记忆关系建模更多依赖LLM本身理解，而非存储层面的图结构。
Claude (闭源，Anthropic)	超大上下文窗口（最高100K tokens）提供超强短期记忆 ￼。一般对话不易触碰上限，可完整保留几十轮甚至上百轮对话内容。Claude会在上下文接近上限时丢弃最旧部分（尤其工具输出等次要内容） ￼来腾空间，尽量确保近期对话不丢失。没有明确的自动摘要功能，大部分场景下无需摘要即可涵盖全部上下文。	正开发Persistent Knowledge Bases：允许用户将资料库与Claude关联，实现跨对话调用 ￼。Claude可对这些文档执行RAG检索，将相关内容融入回答 ￼。企业版Claude引入“项目(Project)”概念，每个项目有独立知识库，Claude记忆可在项目内持久化。一些泄露信息称Claude会有持久内存升级，让模型“自动记住”过去的对话细节，但尚未正式发布 ￼。现阶段Claude默认不保留任一会话的内容到下一会话（除非用户手动提供）。	短期记忆全在模型上下文中处理，不涉及外部存储。长期记忆部分通过Anthropic后台的向量索引实现（推测）。Claude的知识库功能类似插件：由Anthropic托管向量数据库，用户上传文档->系统生成向量->对话时Claude调用检索API获取内容。Claude自己不直接接触存储细节，只负责根据用户请求决定检索哪些内容。	主要靠长上下文代替：Claude因为拥有非常大的窗口，一般不需要对对话进行压缩。不过Anthropic文档提到Claude会自动移除那些久远且不相关的信息以保持注意力 ￼。如果用户提供了过多材料，也建议先让Claude总结重点再深入某部分。未来持久内存推出后，Claude可能在对话结束时自动总结关键信息写入知识库，以备下次调用（类似Progress File思路），但这尚未证实。	暂未：Claude当前以RAG为主，没有公开的知识图谱模块。Anthropic更多通过规则（Constitution）来约束模型行为，而不是通过显式图数据库来存知识。不过其持久记忆扩展可能逐步引入关系概念（例如项目之间共享的概念）。截至2025年，Claude记忆依然是文本型的。
Inflection Pi (闭源)	采用Transformer模型（Inflection-2系列）进行对话，具体上下文长度未知但估计在数千token级别。Pi会尽量保持当前对话的上下文，跨平台同步也确保同一会话在不同设备上连续 ￼。对于超长对话，Pi模型有时表现出遗忘/重复，这提示其短期记忆有限，可能依赖滚动窗口或摘要，但效果有限 ￼。	高度重视跨会话连续性：Pi维护每个用户的持久对话历史和偏好档案 ￼ ￼。不论用户何时回来聊天，Pi都有上下文延续（记得之前谈话内容和情绪基调）。存储上应为云端数据库，内容包括过往聊天记录、提取的用户个人信息（姓名、兴趣）、用户反馈等。Pi会根据新话题检索过去相关的聊天内容，以实现“记得你曾经说过…”的效果 ￼。另外，Pi可能持续更新用户情感曲线（比如这段时期用户常感到压力），从而调整回应方式 ￼。	后端存储估计是专有数据库+嵌入索引。细粒度如何不详，但Toolstack测评提及Pi有“详细的对话记忆和用户偏好同步系统” ￼。这暗示Inflection针对每用户存了结构化的数据：如键值对（用户喜欢的语言=英语），以及整个聊天log的embedding索引。检索时可能先定位用户profile中的明确条目，再通过embedding搜索相似话题的对话记录。	可能有：Pi未透露是否用摘要，但从其长对话重复现象看，自动摘要能力有限 ￼。Pi更多是靠模型本身的对话延续性。如果有，也是后台将非常久远的历史浓缩成“记忆提示”。但用户无法看到明确摘要过程。	否：Pi注重“情感和关系”记忆，但并未公开使用知识图谱库。它记忆谁是用户的重要他人、用户喜好等，这些关系大概以属性形式存储，而非灵活的图数据库查询。暂未见Pi能基于复杂关系网推理（如不会推断“你朋友的孩子叫什么”除非用户自己提供）。
Google Gemini (闭源)	提供百万级token上下文作为短期记忆容器 ￼。模型可在Prompt中同时容纳大量来自用户各数据源的内容（文本、图片描述等）进行综合推理 ￼ ￼。同时引入Context Packing算法动态筛选信息进入上下文，保证在窗口内利用最相关的数据 ￼。因此短期记忆不再是顺序的消息列表，而是由Personal Intelligence引擎从各处挑选拼装的“工作记忆”。	拥有Personal Memory用于跨会话、跨应用积累用户信息 ￼ ￼。Gemini会持续从用户过去的聊天、使用Google服务的记录中学习用户偏好，将其存入个性化档案（例如了解用户的亲友关系、常去地点等） ￼。遇到相关对话时，通过工具查询这些数据源（Gmail、日历、相册等）获取具体细节 ￼。长期记忆还包括对过去聊天的总结，Gemini支持用户查看过去聊天并选哪些用于个性化（可在设置中选择“引用过往聊天”） ￼。企业版提供内网知识连接，Gemini也可从企业文档中检索信息。	存储分布在各Google产品的数据存储中（邮件服务器、照片库、联系人/图谱数据库等）。Gemini的Personal Intelligence Engine负责安全地聚合查询：利用Google账号的各项数据权限检索所需信息 ￼。检索技术包含Google成熟的搜索和密集向量检索（Gemini有专用embedding模型） ￼ ￼。同时Gemini 3模型本身有强工具使用能力，可自主决定调用哪些“插件”获取数据 ￼。可以认为Gemini的存储并非一个单一数据库，而是一个联网的个人数据图谱：不同应用的数据通过用户身份关联，在需求时被调取整合。	支持：Gemini明示采用了“动态上下文拼接”技术 ￼——本质就是一种智能压缩，把超出模型处理能力的庞大个人数据按需取舍，凝炼成模型能消化的内容。相当于实时生成针对当前问题的概要信息供模型使用，而非生硬地截断或全盘传入。此外，Gemini或许也对历史对话进行离线总结，形成用户兴趣模型。总之，总结归纳是Personal Intelligence引擎的重要功能，使Gemini能在海量信息中提炼要点给模型。	部分是：Gemini背后实际构建了用户个人知识图。它能理解诸如家庭关系、预订行程等复杂上下文 ￼——这些信息在系统内部应当以结构化关系表示才能被有效利用。Google在知识图谱方面经验丰富，Gemini很可能将用户数据转成了内部的KG节点，如人物（联系人）、事件（行程）、时间地点等，通过关系连接。Gemini模型通过引擎查询到这些节点信息，再生成回答。因此可说Gemini隐含使用了知识图谱技术来管理记忆。但从模型交互视角看，这对开发者是透明的——开发者只需调用Gemini API，背后的图谱查询由Google基础设施完成 ￼。

**参考来源：**本文调研引用了官方文档、博客和报道等信息源，其中包括LangChain文档【9】【13】【38】、Auto-GPT文档和解析【22】【34】、OpenAI新闻报道【27】、Anthropic公告【39】及社区讨论、Inflection AI 产品评测【5】、Google 技术报告【41】以及关于图谱记忆的技术博客【42】等。上述引用标注格式如【编号†Ln-Lm】对应相应来源的行号，便于查证原文。总体而言，当前AI Agent在记忆机制上各有侧重，但“让AI记住什么、如何高效地记住”仍是演进中的挑战。随着上下文图谱等新技术的融入，我们有望看到更聪明、更善解人意的智能体持续涌现。 ￼ ￼

# Agent memory systems are transforming how AI reasons and persists knowledge

The AI industry has converged on a multi-tier memory architecture that mirrors human cognition, combining **short-term working memory** (in-context), **long-term storage** (vector databases and graphs), and specialized memory types for episodes, facts, and procedures. The critical insight of 2024-2025: memory is no longer optional for production agents—it's the infrastructure that enables continuity, personalization, and multi-step reasoning. Context graphs have emerged as a powerful complement to vector databases, enabling relationship-aware retrieval and temporal reasoning that pure semantic similarity cannot provide.

## Five memory types form the cognitive foundation of AI agents

Modern agent memory systems draw directly from cognitive science, implementing five distinct memory types that serve different purposes in agent reasoning.

**Short-term (working) memory** operates within the LLM's context window—typically **4K to 200K tokens**—and holds the agent's current cognitive workspace: system instructions, recent conversation, and active task state. MemGPT pioneered treating this like computer RAM, with automatic "paging" between context and external storage when limits are reached. The key innovation is self-editing memory: agents use function calls like `core_memory_replace()` to modify their own context, achieving **85-93% token reduction** compared to full-context approaches.

**Long-term memory** persists across sessions through external storage. The industry has settled on a hybrid approach combining vector databases for semantic search with structured storage for metadata. OpenAI's ChatGPT now references **all past conversations** for Plus/Pro users (launched April 2025), while Anthropic stores memories in transparent Markdown files (CLAUDE.md) that users can directly edit and export.

**Episodic memory** captures specific experiences with temporal markers. Stanford's Generative Agents paper established the canonical implementation: a "memory stream" where each entry contains a natural language description, timestamp, importance score, and embedding. Retrieval uses a scoring function balancing **recency, importance, and relevance**. The system demonstrated emergent behaviors like party planning and relationship formation across 25 simulated agents.

**Semantic memory** stores facts and knowledge independent of when they were learned. GraphRAG (Microsoft Research, 2024) advanced this significantly by creating knowledge graphs from document corpora, using community detection to cluster related concepts, and generating pre-computed summaries for efficient retrieval. This enables answering "global" questions like "What are the main themes?" that traditional RAG struggles with.

**Procedural memory** stores skills and how-to knowledge. Voyager (2023) established the pattern of storing skills as executable code with descriptions in a searchable library. The agent retrieves relevant skills via semantic search and composes them for new tasks, achieving **3.3× more unique items** and **15.3× faster milestone unlocking** than baselines in Minecraft. MACLA (2025) extended this with Bayesian selection and contrastive refinement, reaching **90.3% accuracy** on unseen ALFWorld tasks.

## Major AI companies have diverged significantly in memory architecture

OpenAI, Anthropic, Google, Microsoft, and Meta have each taken distinct approaches to agent memory, reflecting different philosophies about personalization, privacy, and user control.

**OpenAI's ChatGPT** operates a dual-layer system: explicit "saved memories" that users can view and edit, plus automatic reference to all past conversations. The April 2025 upgrade means ChatGPT maintains long-term understanding of user preferences, working style, and context without explicit memory requests. Memories are stored as discrete items included in each prompt's context, with the model autonomously managing updates and consolidation.

**Anthropic's Claude** takes a transparency-first approach with project-scoped Markdown files. Memory is organized hierarchically (Enterprise → Project → User → Repository), and the entire curated memory document loads into Claude's **200K-token context window**. Users can export memories to other systems—a unique interoperability feature. Claude 4 models can autonomously create and maintain "memory files" when given file access, demonstrated by creating navigation guides while playing Pokémon.

**Google's Gemini** leverages massive context windows (**up to 2 million tokens** in Gemini 1.5 Pro) combined with "Personal Intelligence" that integrates across Gmail, Photos, Search, and YouTube. Memory is stored in a structured `user_context` document with timestamps and rationales. Google's research arm contributed ReadAgent, which increases effective context **20×** using "gist memories," and the Titans architecture that prioritizes information by "surprise."

**Microsoft Copilot** stores memories in a hidden folder within users' **Exchange mailboxes**, enabling cross-app learning across Word, Excel, Outlook, Teams, and PowerPoint. This follows Microsoft's existing security and compliance infrastructure (Zero Trust, EU Data Boundary, eDiscovery). The Copilot+ PC "Recall" feature creates local photographic memory using NPUs with **40-80 TOPS** processing power.

**Meta** provides framework infrastructure rather than a product feature. Llama Stack's Memory API supports multiple storage backends (vector, key-value, keyword, graph) with a plug-in architecture. Llama 4 Scout offers the industry's longest context at **10 million tokens**, enabling entire codebases or document collections in a single prompt.

## Open-source frameworks offer diverse architectural patterns

Six major frameworks dominate the open-source agent memory landscape, each with distinct design philosophies.

**MemGPT/Letta** introduced the "LLM as Operating System" paradigm that inspired much subsequent work. Its two-tier architecture separates main context (always available) from archival memory (searchable via tools). The agent actively manages its own memory through tool calls, deciding when to store, retrieve, or forget information. PostgreSQL serves as the primary backend with vector storage for archival memory. Benchmarks show **94.8% accuracy** on Deep Memory Retrieval tasks.

**LangChain** is transitioning from legacy memory modules (ConversationBufferMemory, ConversationSummaryMemory) to LangGraph's persistence layer. The modern approach uses checkpointers and state graphs rather than explicit memory objects. LangChain remains the most widely adopted framework, with extensive integrations across vector databases, but requires careful context window management to avoid token overflow.

**LlamaIndex** redesigned its memory system in 2024 around "Memory Blocks"—modular components with priority levels. StaticMemoryBlock holds fixed information, FactExtractionMemoryBlock uses an LLM to extract facts, and VectorMemoryBlock stores embeddings. The system automatically manages token budgets through priority-based truncation, evicting lower-priority content first.

**CrewAI** provides the simplest multi-agent memory sharing with a single `memory=True` parameter enabling four memory types: short-term (ChromaDB), long-term (SQLite3), entity memory, and contextual memory. All agents in a crew share memory automatically, with support for external providers like Mem0.

**AutoGen** (Microsoft) uses an actor-based architecture where memory can be treated as a specialized agent. The v0.4 redesign supports asynchronous, event-driven memory operations with ChromaDB integration and state serialization for persistent multi-session agents. It's the most enterprise-ready foundation but requires more manual configuration.

**AutoGPT** implements straightforward short-term/long-term memory with multiple vector database backends (Redis, Pinecone, Weaviate, Milvus). It remains useful for autonomous task execution but is less suitable for production due to reliability issues with loop detection and context management.

## Context graphs add relational intelligence to agent memory

"Context graph" refers to **knowledge graphs specifically engineered for AI consumption**, with optimizations for token efficiency, relevance ranking, provenance tracking, and hallucination reduction. They represent a distinct layer between raw data and LLM context windows.

The term has two emerging interpretations. The technical view sees context graphs as dynamic, query-driven subgraph extractions that fit within LLM context windows—typically **10-100 entities per query** rather than entire knowledge graphs. The strategic view (Foundation Capital's thesis) emphasizes "decision traces"—recording not just facts but why decisions were made, creating searchable precedent for autonomous agents.

**Zep/Graphiti** represents the current state-of-the-art. Built on Neo4j, it implements a bi-temporal model tracking both when events occurred and when they were ingested. The architecture includes three subgraphs: episodes (raw events), semantic entities (extracted relationships), and communities (clustered summaries). Benchmarks show **94.8% accuracy** on Deep Memory Retrieval versus MemGPT's 93.4%, with **90% latency reduction** on LongMemEval. Critically, retrieval requires no LLM calls—hybrid search combines semantic embeddings, BM25, and graph traversal with **~300ms P95 latency**.

**TrustGraph** offers a "Context Graph Factory" for building AI-optimized knowledge graphs with both schema-free GraphRAG and schema-driven Ontology RAG approaches. Its focus is hallucination reduction through grounded context, with features like token budgeting and modular multi-tenant graphs.

**Microsoft's GraphRAG** creates knowledge graphs from document corpora, uses Leiden algorithm clustering to identify communities, and generates pre-computed summaries. This enables answering complex "global" questions that traditional vector RAG fails on, like summarizing main themes across an entire corpus.

The critical distinction from vector databases: vectors find "semantically similar" content through embedding proximity, while graphs preserve explicit typed relationships enabling multi-hop reasoning. For questions like "Find all things that caused X," graphs traverse explicit causal links rather than relying on embedding similarity. The emerging best practice combines both: vector search narrows candidates, graph traversal returns relationship context.

## Storage mechanisms span vectors, graphs, and hybrid approaches

The infrastructure layer for agent memory has matured significantly, with clear use cases emerging for each technology category.

**Vector databases** remain the dominant storage mechanism for semantic memory. Pinecone leads in production deployments with **sub-50ms latency at billion-scale**, fully managed with zero DevOps overhead. Chroma dominates prototyping with its NumPy-like API and zero-configuration setup. Weaviate offers unique hybrid capabilities combining dense vectors with sparse BM25 search and knowledge graph features. Qdrant excels at complex metadata filtering alongside vector similarity. Milvus handles massive scale with GPU acceleration.

**PostgreSQL + pgvector** has emerged as the practical choice for teams with existing PostgreSQL infrastructure, offering hybrid queries combining vector similarity with SQL filters. HNSW indexing and half-precision vectors reduce memory requirements, though performance lags purpose-built vector engines at scale.

**Redis** provides ultra-low latency (**single-digit milliseconds**) for real-time memory applications, functioning as both semantic cache and session manager. Its in-memory architecture makes it ideal for high-throughput scenarios where latency matters more than storage cost.

**Knowledge graphs** (Neo4j, Amazon Neptune) provide the relational intelligence that vectors lack. Neo4j powers Zep/Graphiti's temporal memory architecture, with native support for vector and BM25 indexes alongside graph queries. The Graph Data Science library enables analytics like community detection and PageRank for memory organization.

The hybrid pattern combining multiple storage types has become standard for production systems. Zep uses Neo4j for graphs plus vector embeddings plus BM25 for full-text search. MemGPT pairs PostgreSQL for core state with vector storage for archival memory. Mem0 offers graph memory backed by Neo4j that extends its vector foundation. This multi-tier approach mirrors cognitive science models of human memory and enables sophisticated behaviors including reflection, planning, and cross-session personalization.

## Recent research advances memory beyond simple retrieval

Academic research in 2024-2025 has pushed agent memory from simple RAG toward sophisticated cognitive architectures.

**A-MEM** (February 2025) introduced Zettelkasten-inspired memory organization, storing memories as interconnected notes with contextual descriptions, keywords, tags, and embeddings. Dynamic linking analyzes the repository to establish semantic connections, doubling performance on multi-hop reasoning while achieving **85-93% token reduction**. Processing takes approximately **5.4 seconds** with GPT-4o-mini or **1.1 seconds** with local Llama 3.2.

**HippoRAG** (NeurIPS 2024) draws on hippocampal indexing theory, using knowledge graphs as an associative index analogous to the hippocampus, with the LLM serving as the neocortex for encoding and retrieval. Personalized PageRank traverses the graph during retrieval. The approach achieves **20% improvement** on multi-hop QA while being **10-20× cheaper** and **6-13× faster** than iterative retrieval methods.

**Mem0** (2025) demonstrated production-ready scalable memory with **26% improvement** over OpenAI on LLM-as-a-Judge metrics, **91% lower P95 latency**, and **90%+ token cost savings** versus full-context approaches. Its graph-based variant adds relational structure for complex reasoning scenarios.

**Agentic Memory (AgeMem)** (January 2026) integrates memory management directly into agent policy through reinforcement learning. Memory operations (STORE, RETRIEVE, UPDATE, DELETE, SUMMARIZE, FILTER) are exposed as tool-based actions, with three-stage progressive training that teaches long-term construction, short-term control under distractors, and full task coordination.

Survey papers have synthesized the field's rapid evolution. "Memory in the Age of AI Agents" (December 2025) distinguishes three memory forms (token-level, parametric, latent) and three functions (factual, experiential, working). Key research frontiers include memory automation, RL integration, multimodal memory for images and video, multi-agent memory sharing, and trustworthiness concerns around privacy and alignment.

## Context graphs and memory systems are complementary, not competing

The relationship between context graphs and memory systems resolves to complementarity rather than substitution. Context graphs are a **component within** broader memory architectures—specifically, they provide the structured relationship layer that enables multi-hop reasoning and temporal queries.

A typical production architecture combines: short-term memory (conversation buffer and working context), long-term memory with both vector stores for semantic search and context graphs for relationship modeling, a summarization layer for compressed context, and a retrieval/assembly layer that selects and structures information for the LLM.

Vector databases answer "what's similar?" while context graphs answer "what's connected?" and "how did this change over time?" For applications requiring only semantic similarity—like finding relevant documentation—vectors suffice. For applications needing relationship reasoning, temporal tracking, or decision traces—like enterprise assistants or autonomous agents—context graphs become essential infrastructure.

The emergence of "context engineering" as a discipline reflects this synthesis. As Neo4j frames it: "Context engineering has moved from clever prompts to disciplined systems that deliver facts, memory, and tools at the right moment." The goal is selecting precisely what the model needs—not too much, not too little—structured in a form that enables effective reasoning. Context graphs, vector databases, summarization, and episodic memory all serve this goal through different mechanisms.

## Conclusion

Agent memory has matured from experimental feature to essential infrastructure in 2024-2025. The field has converged on cognitive science-inspired taxonomies (short-term, long-term, episodic, semantic, procedural) while diverging on implementation approaches across major AI companies. Context graphs have emerged as the key innovation for relationship-aware, temporally-grounded memory—complementing rather than replacing vector-based semantic search. The hybrid pattern combining multiple storage mechanisms (vectors for similarity, graphs for relationships, SQL for structured data) has become the production standard.

Three critical trends will shape the next phase: first, agents learning to manage their own memory through reinforcement learning rather than hand-crafted rules; second, multimodal memory extending beyond text to images, video, and structured data; and third, multi-agent memory architectures enabling shared knowledge and coordinated learning across agent teams. The infrastructure exists—the challenge now is building agents sophisticated enough to use it effectively.

目前使用 Context Graph（上下文图/知识图谱）来规划 Memory 的工具和框架正在快速爆发。

根据它们在架构中的角色，可以将其分为三类：**开发框架 (Frameworks)**、**专用记忆服务 (Memory Services)** 和 **底层数据库 (Infrastructure)**。

### 1. 核心开发框架 (Frameworks)

这些是开发者构建 Agent 的“骨架”，它们提供了现成的模块来让 Agent “长出”图记忆。

* **LangChain / LangGraph (开源 - 极热门)**
* **如何使用:** LangChain 提供了与 Neo4j 和 FalkorDB 的深度集成，允许开发者将对话历史写入图数据库。
* **规划 (Planning) 特性:** **LangGraph** 本身就是一个基于“图”的控制流框架。它不仅仅是用图存数据，而是**用图来规划 Agent 的思考路径**（StateGraph）。
* *区别:* 它用图来定义“状态”和“转换”（例如：规划 -> 执行 -> 检查），这是一种“流程图”式的记忆规划。


* **最新动态:** 推出了 `LangGraph Memory`，并支持与持久化层（Checkpointers）结合，社区中大量案例使用 Neo4j 作为其长期记忆后端。


* **LlamaIndex (开源 - 专注于数据)**
* **如何使用:** 它是目前对 **GraphRAG** 支持最激进的框架。它有专门的 `KnowledgeGraphIndex` 和 `PropertyGraphIndex`。
* **核心功能:** 它能自动从你的文档或对话中抽取实体（Entity）和关系（Relation）构建图谱，并在 Agent 回答问题时，先在图上进行“多跳推理”检索，再喂给 LLM。
* **适用场景:** 适合需要处理大量私有数据、且数据间关系复杂（如法律、金融）的 Agent。


* **Microsoft GraphRAG (开源 - 研究级)**
* **地位:** 这是一个由微软研究院推出的**参考实现**，而非通用的 Agent 框架，但它是目前业界模仿的标杆。
* **核心逻辑:** 它不只是检索节点，而是利用算法（如 Leiden 算法）将图划分为不同的“社区（Communities）”，并生成社区摘要。
* **记忆规划:** Agent 在回忆时，是由“宏观（社区摘要）”到“微观（具体节点）”进行规划检索的。



### 2. 专用记忆服务 (Memory Services)

这些工具旨在成为 Agent 的“外挂大脑”，你只需通过 API 调用它们，它们会自动帮你维护 Context Graph。

* **Zep (开源/闭源混合)**
* **核心技术:** 推出了名为 **Graphiti** 的引擎。
* **特点:** 这是一个**时序知识图谱 (Temporal Knowledge Graph)**。
* **记忆规划:** 它不仅记住“事实”，还记住“事实发生的时间”。例如，它能区分“用户**去年**喜欢吃苹果”和“用户**现在**对苹果过敏”。它能自动将非结构化的聊天记录转化为图结构，是目前体验最接近“即插即用”的图记忆产品。
* **对比:** 它在官方评测中声称比 MemGPT 的检索准确率更高。


* **Mem0 (开源/云服务)**
* **定位:** "The Memory Layer for AI Agents"。
* **特点:** 它是一个混合系统，结合了**向量检索**（模糊匹配）和**图存储**（结构化关系）。它会根据用户输入自动提取偏好（Preferences）并建立关联，虽然它主要宣传是“个性化记忆”，但底层正在积极引入 Graph 结构来解决冲突问题。


* **Graphlit (闭源/商业)**
* **定位:** 专为 AI 应用设计的“无服务器数据平台”。
* **特点:** 它原生就是基于知识图谱构建的。它会自动摄取文件、网页、Notion 等数据，构建图谱，并为 Agent 提供“内容 + 记忆”的检索 API。它比单纯的向量数据库更“懂”数据间的关联。



### 3. 特殊提及：MemGPT (现更名为 Letta)

* **状态:** 虽然 **Letta (MemGPT)** 非常热门，但它**原生并不基于 Context Graph**。
* **原理:** 它的设计灵感来自“操作系统”，使用的是分级内存（Core/Recall/Archival），本质上主要是文本块和向量检索。
* **新动向:** 社区正在积极探索将 Letta 与 Neo4j 结合（有相关 GitHub Discussions），让它的“长期归档存储”变成一个图谱，而不是简单的数据库。

### 总结：应该怎么选？

| 你的需求 | 推荐工具 | 核心逻辑 |
| --- | --- | --- |
| **我要构建复杂的 Agent 流程**，需要它按步骤思考 | **LangGraph** | **控制流图** (Control Flow Graph)。用图来规划“行动步骤”。 |
| **我要 Agent 记住复杂的用户关系**（如谁是谁的朋友，发生了什么） | **Zep** 或 **LlamaIndex + Neo4j** | **数据图** (Data Graph)。用图来存储“事实与关系”。 |
| **我只是不想写后端代码**，想要一个现成的 API | **Mem0** 或 **Zep Cloud** | 托管式记忆服务 (Managed Memory)。 |
| **我想做科研或极高精度的检索** | **Microsoft GraphRAG** | 社区发现与层级摘要。 |

**简而言之：** 目前业界正在从单纯的“向量数据库”向“向量+图”的混合模式转变。**Zep** 和 **LlamaIndex** 是目前在这个方向上跑得最快的两个实用工具。