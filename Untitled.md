# 

## **1\. 调研对象与比较口径**



### **1\.1 固定项目身份，避免同名混淆**



|工具|本文对应项目|核查分支与提交短号|主要定位|
|---|---|---|---|
|CodeGraph|colbymchenry/codegraph|main，ba3c21e5|本地代码关系索引、源码探索与 MCP 查询|
|GitNexus|abhigyanpatwari/GitNexus|main，d2a43e33|仓库知识图谱、模块与流程探索、AI 助手|
|Codebase\-Memory|DeusData/codebase\-memory\-mcp|main，59a05eb1|本地持久化代码图谱、持续索引、MCP 与管理界面|
|Graphify|Graphify\-Labs/graphify|v8，26b02b5e|代码与文档等多源内容融合的知识图谱|



项目入口：[CodeGraph](https://github.com/colbymchenry/codegraph)、[GitNexus](https://github.com/abhigyanpatwari/GitNexus)、[Codebase\-Memory](https://github.com/DeusData/codebase-memory-mcp)、[Graphify](https://github.com/Graphify-Labs/graphify)。提交快照保存在随附 four\_tools\_sources 目录；后续版本可能改变能力。



这里把“代码知识图谱工具”作为工程比较口径：工具显式组织代码实体及其关系，并提供跨实体查询、导航或推理入口。Graphify 还融合非代码知识，范围更宽。AST、调用图和程序依赖图则是重要的构图素材或相关程序图表示，不能不加区分地把所有相关论文都称为完整 CKG 系统。



### **1\.2 你的工程需要解决什么**



按提供的架构图，目标不仅是画出函数之间的连线，还包括 Cargo 工程层级、API 与类型约束、依赖与调用、编译诊断及 HIR/MIR 证据。评价工具时，应问“能否回答这些问题”，而不只问“用了什么图数据库”。



|需求层面|具体问题示例|
|---|---|
|工程结构|当前 workspace 有哪些 package、crate、target 和 module？依赖如何受 feature 与 target 影响？|
|API 与类型|某方法属于哪个 impl？trait 约束和关联类型是什么？调用解析到哪个候选实现？|
|程序分析|某值在哪里移动、借用和释放？相应结论来自哪次编译与哪项分析？|
|证据关联|一条调用边、诊断或 unsafe 标记能否回到源代码，以及对应 HIR/MIR 位置？|
|工程可用性|修改代码或配置后，哪些事实过期？未解析的关系如何展示？|



## **2\. 四个工具如何构建代码知识图谱**



### **2\.1 CodeGraph：语法抽取与规则解析，面向源码关系探索**



**构建链路：**仓库源码 → Tree\-sitter 解析 → 提取符号、导入和关系候选 → 定义/引用及框架规则解析 → SQLite 图索引与全文检索 → CLI、MCP、本地 Web 界面。



它先识别函数、类、方法等实体，再建立调用、导入、继承或实现等关系。跨文件关系需要结合定义和导入信息继续解析，不能把整个过程描述为“只是把 AST 原样存进数据库”。框架专用关系可带 heuristic 来源标记；源码中也保留未解析引用，说明系统允许表达解析不确定性。[构建说明](https://github.com/colbymchenry/codegraph/blob/ba3c21e50d9129d2f5f3843ec3728868ae6d47a1/site/src/content/docs/core-concepts/how-it-works.md)



**图模型与存储。**使用 SQLite 存储节点和边，不影响其作为图谱被查询。节点可以包含类型参数、返回类型等字段；边包含种类、位置和 provenance。边的唯一性考虑关系种类与调用位置，有利于区分同一对符号之间不同的关系或调用点。此处应区分“记录类型文本/解析结果”和“获得编译器完成类型检查后的语义”。\[数据库模式\]\(https://github\.com/colbymchenry/codegraph/blob/ba3c21e50d9129d2f5f3843ec3728868ae6d47a1/src/db/schema\.sql\)



**优点。**本地部署路径直接；符号关系与源码阅读紧密结合；支持持续更新，并为启发式关系保留来源信息。查询和导航不要求先让 LLM 重新理解整个仓库。



**对 Rust 的不足与边界。**尚未核实它导出了 rustc 的借用检查结果或按 Cargo 配置区分的 MIR 分析事实。即使工具内部使用 Rust 编写，也不能据此认为它调用了 Rust 编译器。trait 调用、宏生成符号及条件编译的准确性，需要用 Rust 样例验证，不能只凭“支持 Rust”得出结论。



**论文关联。**未检索到与该指定仓库明确对应的正式论文。可用 CodexGraph、RepoGraph 讨论结构图服务代码理解的路线，但不能称它们是 CodeGraph 的配套论文。



### **2\.2 GitNexus：结构图上增加社区与调用流程组织**



**构建链路：**仓库文件结构 → Tree\-sitter 解析 → 导入、调用和继承等跨文件解析 → 社区检测与调用链处理 → LadybugDB 图存储 → 搜索、图谱探索和 AI 查询。



当前源码包含接收者与返回类型等推断逻辑，复杂度高于单纯按函数名匹配。社区检测使用 Leiden，把连接紧密的符号组织为社区；流程处理从入口附近构造调用链，让用户先看到“一个功能可能涉及哪些步骤”。当前架构采用 LadybugDB；不宜直接沿用旧资料中的 Kuzu 描述。[架构文档](https://github.com/abhigyanpatwari/GitNexus/blob/d2a43e33df7d30cf17d9183beab655c0b915a1c6/ARCHITECTURE.md)、[社区处理源码](https://github.com/abhigyanpatwari/GitNexus/blob/d2a43e33df7d30cf17d9183beab655c0b915a1c6/gitnexus/src/core/ingestion/community-processor.ts)



**LLM 在哪里。**基础代码关系的建立主要依赖解析器与规则；LLM 用于图谱上的问答及知识组织等功能。因此“界面带 AI 聊天”不能成为“用 LLM 建图”的分类依据。



**优点。**从符号关系进一步提供社区和流程，能降低读大仓库的入口门槛；同一索引服务搜索、影响分析及 AI 上下文获取；提供浏览器与本地服务两种使用路径。\[项目说明\]\(https://github\.com/abhigyanpatwari/GitNexus\)



**对 Rust 的不足与边界。**社区的形成依据图结构，不能自动等同于真实业务边界；静态调用链不等于实际执行轨迹或已经验证的可行路径。当前说明中可选 PDG/污点分析的语言覆盖与基础解析覆盖不同，不能把 TS/JS 的分析能力外推到 Rust。仍需核验 Rust 的 cfg、宏、trait 动态分发及所有权事实。\[架构文档\]\(https://github\.com/abhigyanpatwari/GitNexus/blob/d2a43e33df7d30cf17d9183beab655c0b915a1c6/ARCHITECTURE\.md\)



**论文关联。**未检索到该指定工具对应的正式论文。RepoGraph、LocAgent 可解释“为什么结构关系能帮助 Agent 定位代码”；它们不是对 GitNexus 的准确率或性能背书。



### **2\.3 Codebase\-Memory：持续索引、规则语义解析与 MCP 服务**



**构建链路：**源码扫描 → Tree\-sitter 语法抽取 → 导入及类型相关规则解析 → 补充工程关系 → SQLite 持久化 → 社区与结构查询 → MCP、CLI 和本地 Web UI。



除基础代码结构外，系统还提供面向测试、HTTP、配置等关系的扩展，并支持利用 Git 共同修改信息及可选 trace 输入丰富分析。文件变化通过哈希与监视机制更新索引；并非每次对话都让模型重新读完仓库。[项目说明](https://github.com/DeusData/codebase-memory-mcp)



**Rust 支持需要准确描述。**当前版本已有针对 use/module、impl/trait、结构体字段、泛型约束、UFCS 等的解析规则。项目所说的 Hybrid LSP 在这里是自实现的类型解析机制，不能直接理解为启动 rust\-analyzer 或获得 rustc 的完整语义。因此“完全没有 Rust 类型信息”和“已经等价于编译器语义”都不准确。\[版本说明与实现概述\]\(https://github\.com/DeusData/codebase\-memory\-mcp\)



**优点。**可持续维护本地代码图，接口与 Agent 工作流结合紧密；部分关系有置信度或解析策略信息；对 Rust 已做专门规则补充，适合用作原型对照基线。



**对 Rust 的不足与边界。**规则覆盖与编译器语义一致性仍需分别评估；置信度数值不等于经过校准的正确概率。需要实验检查宏展开、条件编译、泛型方法和动态分发。其公开论文主要评估 Agent 代码探索任务，并没有证明完整的 Rust 借用语义建模能力。



**直接相关论文。**《Codebase\-Memory: Tree\-Sitter\-Based Knowledge Graphs for LLM Code Exploration via MCP》，2026 年预印本。论文评估的版本早于本次源码快照，论文结果不能直接当成当前版或 Rust 专项测试结果。\[论文全文\]\(https://arxiv\.org/html/2603\.27277v1\)



### **2\.4 Graphify：代码结构与文档等多源知识融合**



**构建链路：**代码经 Tree\-sitter 与语言规则抽取；文档、PDF、图片等经语义抽取，音视频先转录 → 合并实体和关系 → NetworkX 图处理与社区检测 → 图 JSON、HTML 交互图及报告等导出。



正常代码抽取路径不需要 LLM；模型主要参与非代码内容的语义抽取等环节。Rust 提取器识别函数、结构体、枚举、trait、impl、use 等，并包含调用关系处理。因此不应把 Graphify 说成“把所有源码直接交给 LLM 生成图”。[构建说明](https://github.com/Graphify-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/docs/how-it-works.md)、[Rust 提取器](https://github.com/Graphify-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/graphify/extractors/rust.py)



**图模型与来源。**关系可标记 EXTRACTED、INFERRED、AMBIGUOUS，帮助区分抽取与推断；INFERRED 并不必然来自 LLM，也可能来自确定性的二次解析。当前经 NetworkX Graph/DiGraph 的聚类组图路径使用简单图模型；其中 DiGraph 在同一有序节点对上只能保存一条边。虽然代码有关系覆盖保护，但多个不同关系仍有折叠风险。这一判断针对该组图路径；绕过 NetworkX 的原始边列表处理路径不同，不能概括为所有输出都丢失关系。\[组图实现\]\(https://github\.com/Graphify\-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/graphify/build\.py\)



**优点。**更容易把“代码怎么实现”与“设计文档为什么这样规定”关联起来；输出可移交的图文件和报告；有缓存、更新与监视路径，适合研究资料探索及代码知识整理。\[架构\]\(https://github\.com/Graphify\-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/ARCHITECTURE\.md\)



**对 Rust 的不足与边界。**文档中的解释关系不能替代编译器事实；跨来源实体合并仍可能造成歧义。Rust 分析可能要求在相同实体之间同时保留 calls、borrows、moves 等关系，还要区分位置和构建配置，上述简单图组图路径不适合直接承载全部此类事实。



**论文关联。**未检索到与该指定仓库明确匹配的代码知识图谱论文。LogicLens 可作为相近路线；题名含 Graphify、讨论 GraphQL/Gremlin 的同名论文不属于这个项目，不应引用为配套论文。



## **3\. 从构建方式分类：两类主路线，多个可叠加环节**



**对这四个工具，建议采用两类主路线，而不是强行把四个工具分成四类。**这只是当前样本的归类，不代表所有代码知识图谱只有两种构建方法。



|主路线|核心做法|本次代表|主要优势|主要代价|
|---|---|---|---|---|
|A\. 代码静态解析与规则解析主导|从语法树提取实体，再用导入、符号、类型和框架规则解析关系|CodeGraph、GitNexus、Codebase\-Memory；Graphify 的代码子流程也属于此类|可复现、可追溯到源码，通常不要求完整构建；适合持续索引|规则有覆盖边界，复杂语义的正确性依赖语言专项验证|
|B\. 代码结构与多源语义融合|保留代码结构，同时对文档等抽取概念、描述及跨来源关系|Graphify|能联系代码、设计意图与业务知识|实体对齐更难，语义推断必须与程序事实分开|



几个容易混淆的维度应单独列：SQLite/LadybugDB/NetworkX 属于存储与处理选择；Leiden/Louvain 属于图上组织方法；MCP 属于访问协议；AI 问答属于消费方式。它们本身都不能证明图谱拥有更深的程序语义。



更广泛的相关工作还存在编译器或静态分析驱动的构图，以及运行轨迹、版本历史等数据补充路线。本文四工具中可见这些补充环节，但没有充分证据将某个工具整体归为“基于 rustc MIR 的 Rust 完整语义图谱”。“完整程序语义图谱”也不是适宜轻易承诺的目标：静态分析本身存在近似和不可判定性边界。



## **4\. 相关论文：用来解释设计取舍，而不是给工具贴论文标签**



以下 8 篇是本次汇报最相关的重点阅读。只有 P1 与本次指定工具直接对应；其余是同类方法或相关程序图研究。2026 年工作按预印本对待。此前整理的 \[15 篇分类文献文档\]\(semantic\_depth\_papers\.docx\) 和 \[21 项综合调研\]\(report\.html\) 可作扩展阅读，本次汇报以这里明确限定的对应关系为准。



### **P1｜Codebase\-Memory，2026，直接相关工具论文**



完整题名：Codebase\-Memory: Tree\-Sitter\-Based Knowledge Graphs for LLM Code Exploration via MCP。



**初学者理解：**先给仓库制作一张可以反复查询的“关系地图”，Agent 提问时从地图取相关代码，而不是每次从零搜索全部文件。论文用 Tree\-sitter、跨文件解析、持久化图与 MCP 工具实现这个过程。



**证据与价值：**作者在 31 个仓库上评估代码探索，报告了更低的 token 与工具调用开销，但质量并未全面超过对照。评估版本为较早版本，评分方法和作者参与评估等因素限制外推；这些结果衡量的是探索任务表现，不是调用边精确率，更不是 Rust 借用分析正确率。



**用于本项目：**借鉴“结构预计算换取查询效率”，同时把 Rust 关系正确率与 Agent 问答质量分成两套指标。\[全文\]\(https://arxiv\.org/html/2603\.27277v1\)



### **P2｜CodexGraph，NAACL 2025，代码图数据库与 Agent**



完整题名：CodexGraph: Bridging Large Language Models and Code Repositories via Code Graph Databases。



**初学者理解：**把模块、类、函数等做成数据库中的实体，Agent 将找代码的问题改写为图查询。它可以先找类，再沿继承、成员或使用关系找到相关实现，减少盲目阅读。



**方法与取舍：**通过静态结构分析建立图，LLM 负责生成与使用查询，而不是凭空生成基础代码关系。优势是查询可组合、结果可追溯；局限是查询能回答什么取决于预先定义的模式与分析覆盖。论文并不证明该模式足以表达 Rust move/borrow。



**关联工具：**有助于解释四工具的查询层与 MCP/Agent 使用价值，尤其应借鉴其显式图模式思想。\[正式论文页面\]\(https://aclanthology\.org/2025\.naacl\-long\.7/\)、\[全文\]\(https://arxiv\.org/html/2408\.03910v2\)



### **P3｜RepoGraph，ICLR 2025，仓库级程序图**



完整题名：RepoGraph: Enhancing AI Software Engineering with Repository\-level Code Graph。



**初学者理解：**处理缺陷时，某行代码附近的文本不一定最重要；真正相关的代码可能在它调用的另一个文件里。RepoGraph 用定义、引用与调用等关系帮助 Agent 取得这种跨文件上下文。



**方法与取舍：**使用 Tree\-sitter 和仓库级关系组织代码，围绕检索目标取邻域。结构关系能补充向量检索，但源码关系不等于完整执行语义。为聚焦仓库内部而过滤外部库，是一种任务取舍；你的工程若要展示 crate 依赖与外部 API，就不能照搬这种范围裁剪。



**关联工具：**可解释 CodeGraph、GitNexus 和 Codebase\-Memory 的结构上下文价值；它属于相关仓库图工作，不能直接等同为四工具的实现依据。\[全文\]\(https://arxiv\.org/html/2410\.14684\)



### **P4｜LocAgent，ACL 2025，图引导代码定位**



完整题名：LocAgent: Graph\-Guided LLM Agents for Code Localization。



**初学者理解：**先把“问题可能在哪个目录和文件”逐步缩小到“哪个类和函数”，再沿调用等关系检查周边代码。图提供路线，Agent 决定下一步看哪里。



**方法与取舍：**建立目录、文件、类、函数及包含、导入、继承、调用关系，结合检索与图探索定位相关代码。优势是把结构搜索做成可操作的定位流程；目标是定位任务，不是验证所有权安全，也不是构建最完整的程序模型。



**关联工具：**适合支撑 UI 中“从搜索结果进入局部关系、再打开源码”的设计，以及 MCP 工具的粒度划分。\[正式论文与全文入口\]\(https://aclanthology\.org/2025\.acl\-long\.426/\)



### **P5｜GraphGen4Code / Graph4Code，2020 起，静态分析与多源关联**



完整题名：A Toolkit for Generating Code Knowledge Graphs。



**初学者理解：**代码里的 API 用法是一部分知识，API 文档与开发者讨论是另一部分。该工作试图把这些知识接在一起，使查询既能找到程序关系，也能找到解释材料。



**方法与取舍：**利用 WALA 等静态分析获得程序关系，结合文档、讨论及 RDF 表示组织知识。优势是将分析得到的事实与外部知识统一关联；代价是分析基础设施、实体链接和来源管理更复杂，且分析框架覆盖不能自然迁移为 Rust 语义覆盖。



**关联工具：**与 Graphify 的多源方向可比较，但语义来源不同：静态数据/控制流分析不能由自然语言描述替代。\[论文全文\]\(https://arxiv\.org/pdf/2002\.09440\)



### **P6｜LogicLens，2026 预印本，多仓库语义代码图**



完整题名：LogicLens: Leveraging Semantic Code Graph to explore Multi Repository large systems。



**初学者理解：**仅知道 A 调用了 B，通常还不能解释“用户登录是怎样完成的”。该工作在代码结构上增加自然语言语义和领域概念，让用户跨仓库理解功能。



**方法与取舍：**结合语法解析和 LLM 语义加工，连接程序结构与功能解释。优势是对系统理解和业务问答更友好；局限是高层语义需要核验，而且论文所覆盖的语言与评估不能直接证明 Rust 泛型、借用分析能力。



**关联工具：**是 Graphify 的相关路线论文，而非 Graphify 的配套论文；提醒我们分别保存“程序证据”和“功能解释”。\[全文\]\(https://arxiv\.org/html/2601\.10773\)



### **P7｜Code Property Graph，IEEE S\&P 2014，深层程序图基础**



完整题名：Modeling and Discovering Vulnerabilities with Code Property Graphs。



**初学者理解：**AST 告诉你代码写了什么，控制流告诉你可能按什么顺序执行，数据依赖告诉你一个值从哪里来。把这几类关系合起来，才能表达许多跨语句的漏洞模式。



**方法与取舍：**在统一图表示中结合 AST、CFG 和 PDG，以图遍历查询漏洞模式。优势是关系更接近程序行为；代价是分析与查询更复杂，且仍受静态分析近似限制。原工作不是 Rust 图谱，也不等价于“经过完整语义证明的知识图谱”。



**用于本项目：**论证为什么仅有调用边不足以回答 move/borrow 问题；不能据此宣称已经有开箱即用的 Rust MIR 全语义方案。\[论文全文\]\(https://www\.ieee\-security\.org/TC/SP2014/papers/ModelingandDiscoveringVulnerabilitieswithCodePropertyGraphs\.pdf\)



### **P8｜Reliable Graph\-RAG for Codebases，2026 预印本，构图可靠性比较**



完整题名：Reliable Graph\-RAG for Codebases: AST\-Derived Graphs vs LLM\-Extracted Knowledge Graphs。



**初学者理解：**同一仓库可以让解析器生成结构图，也可以让 LLM 总结出知识关系；作者比较这些图用于代码问答时的效果与成本。



**证据与边界：**实验涉及 3 个 Java 仓库和合计 45 个查询，AST 与 LLM 方案的模式、关系粒度和覆盖并不完全相同。因此它能说明特定方案组合的取舍，不能证明“所有 LLM 图谱都差于所有 AST 图谱”，也不能直接推广到 Rust。



**用于本项目：**将确定性的结构事实与语义解释分层评估；图关系准确率、证据覆盖率与最终回答质量都要分别测量。\[全文\]\(https://arxiv\.org/html/2601\.08773v1\)



## **5\. 现有图谱的设计优点与不足**



### **5\.1 已有设计值得保留的部分**



|设计|带来的好处|对本项目的启示|
|---|---|---|
|先抽取结构，再提供查询|减少重复阅读仓库，支持可组合的关系检索|将图构建与问答服务分开，允许没有 LLM 时也能工作|
|实体与源码位置关联|用户能够检查结果，错误关系更易排查|每条关键事实关联源码 span、提取器和版本|
|局部邻域与路径探索|比全仓库大图更适合定位与代码阅读|默认展示问题相关子图，允许继续展开|
|社区与流程预计算|给大仓库提供中等粒度入口|保留计算方法和依据，不将聚类直接标记为真实业务模块|
|文件缓存、监视与持续索引|支持日常开发，而非一次性演示|在文件更新之外增加配置与依赖驱动的失效处理|
|图谱与文档融合|支持设计意图、使用说明及业务问答|文档解释层与编译事实层分别标识，再通过证据链接关联|



上述优点来自不同工具和论文的组合，不能认为每个工具都以相同方式实现了全部能力。



### **5\.2 对照 Rust 工程，真正需要补齐什么**



下面“差距”指现有公开材料尚不足以证明满足本项目要求，不等于判定所有工具完全不支持该项。



|Rust 工程要求|四工具已有基础与边界|你的工程应补齐的内容|
|---|---|---|
|Cargo 工程与构建配置|文件、模块、导入图已有基础；不等于 Cargo 的 package、target、feature 与 cfg 解析结果|接入 cargo metadata，并明确 toolchain、target、profile、features 等分析上下文；不同配置的事实不混用|
|trait、泛型与方法绑定|部分工具有类型和 Rust 专用规则，不能概括为纯文本匹配；但规则解析不等于编译器绑定|区分声明关系、候选实现、静态已解析调用和动态分发候选；记录未解析原因|
|宏展开与生成代码|能识别宏调用语法不等于看到展开后的定义与调用|关联宏定义、调用、展开结果和源码映射；说明过程宏及生成代码的覆盖边界|
|所有权、借用与生命周期|类型文本、引用符号和生命周期注解只能提供部分线索|基于合适的 rustc MIR 分析阶段提取 move、borrow、drop 和区域约束等事实；标记分析阶段与证据|
|控制流、数据流及调用|静态调用图与入口调用链有用；不能当作执行轨迹或可行性证明|按具体分析构建 CFG、数据依赖和调用候选；外部函数、FFI 与动态行为明确保留边界|
|编译诊断与多表示关联|源码位置通常可查；未核实统一覆盖 Cargo 配置、诊断、HIR 与 MIR 的证据链|诊断关联源位置、构建上下文和相应语义实体；HIR/MIR 映射允许一对多和无法精确对应|
|多种关系与多处证据|CodeGraph 保留关系种类和位置；Graphify 的简单图组图路径有平行关系折叠风险|支持多重边或将关系实例建成节点；以关系类型、调用点、配置和来源区分事实|
|增量一致性与版本|四工具已有更新机制，不能写成“都不支持增量”|检查 trait、宏、依赖、配置变化对其他文件的影响；展示索引版本、过期状态与失败范围|



Cargo 元数据是工程结构的重要来源，但也不能把一份 metadata 输出当成全部编译语义。rust\-analyzer 有自己的 HIR 抽象，其内部表示不能直接等同于 rustc HIR/MIR。rustc 借用检查以 MIR 为基础；“能够读 MIR”也不自动意味着已经提取了所有借用、区域和数据流结论。[Cargo metadata](https://doc.rust-lang.org/cargo/commands/cargo-metadata.html)、[rust\-analyzer 架构](https://rust-analyzer.github.io/book/contributing/architecture.html)、[rustc 借用检查说明](https://rustc-dev-guide.rust-lang.org/borrow_check.html)



### **5\.3 汇报时可用的具体例子**



**例一：条件编译。**同一个函数调用位于 feature 控制的分支中。语法工具可以同时看见不同分支；你的工程应说明“这条边在哪个构建配置下有效”，而不是把互斥配置的关系混成一个看似确定的程序。无需枚举所有 feature 组合，先对用户选择的配置建立明确语义即可。



**例二：trait 调用。**源码出现 x\.run\(\) 时，应区分已知具体类型上的方法、泛型约束下的调用，以及 dyn Trait 的动态分发。把它们全部画成一条无条件 CALLS 边，会掩盖确定性差别。



**例三：借用。**看到 \&mut T 或显式生命周期参数，只能说明代码包含相应语法或签名信息；它不能单独回答某个值在某个程序点能否被移动。后者需要程序点、数据流与借用约束等分析证据。



**例四：unsafe。**标记 unsafe 所在位置很有用，但它不等于发现漏洞，也不等于验证安全。图谱需要区分语法位置、分析告警与已经核实的问题。



### **5\.4 可直接用于相关工作的表述**



“现有社区代码图谱工具普遍以语法解析和跨文件规则解析为基础，服务于符号导航、结构查询和 Agent 上下文检索；部分系统进一步融合文档语义、社区结构及调用流程。其优势在于轻量构建、交互便利和较好的源码可追溯性。面向 Rust 工程，仅依靠上述结构关系仍不足以保证构建配置、宏展开、trait 绑定与借用约束的一致表达。因此，本项目拟在现有结构图谱能力基础上，引入 Cargo 工程上下文与编译分析证据，并建立可追溯的多层语义关联。”



这段表述可以由前述工具资料及 P1—P8 支撑；不要进一步写成“现有工具均不支持 Rust”或“本项目首次实现完整 Rust 知识图谱”，后者需要更广泛的检索和实验证据。



## **6\. 四工具的用户界面调研**



### **6\.1 CodeGraph：以符号和源码为中心的多视图 Web 界面**



**入口与形态：**本地执行 codegraph ui，提供 Web 查看器。不要因为主 README 展示较少就认为它只有 CLI。独立组件库的发布状态与现有本地查看器是两回事。\[UI 文档\]\(https://github\.com/colbymchenry/codegraph/blob/ba3c21e50d9129d2f5f3843ec3728868ae6d47a1/ui/README\.md\)



\!\[CodeGraph 官方 Symbol 界面：调用者、源码与被调用者并列\]\(four\_tools\_sources/codegraph/assets/codegraph\-ui\-symbol\-view\.png\)



截图来源：[项目官方 Symbol 界面图](https://github.com/colbymchenry/codegraph/blob/ba3c21e50d9129d2f5f3843ec3728868ae6d47a1/assets/codegraph-ui-symbol-view.png)，随 UI 文档提供。截图所示标签可能早于当前源码；以下视图清单同时参考本次核查的 TopBar，而不是仅从截图推断。



**大颗粒功能：**符号搜索与导航；源码和文件阅读；调用者/被调用者联动；模块 Map；关系 Flow 与探索路径；入口点浏览。当前源码还包含 Screens、Steps、Dead code 等入口，其中部分依赖项目内容及可用数据。支持保存探索轨迹，以及部分图视图的图片导出。\[导航实现\]\(https://github\.com/colbymchenry/codegraph/blob/ba3c21e50d9129d2f5f3843ec3728868ae6d47a1/ui/src/components/TopBar\.svelte\)



**交互特点与局限：**把源码与局部关系并排，适合回答“这个符号是什么、谁调用它、它调用谁”。对你的工程，可借鉴该布局，但要新增构建配置、语义来源和不确定性；这些信息不是现有调用连线天然具备的。未核实存在集成于该 Web 查看器的通用 AI 聊天面板，Agent 查询应与 MCP 能力分开描述。



### **6\.2 GitNexus：模块/流程探索与 AI 交互并置**



**入口与形态：**提供浏览器应用，也可连接本地 gitnexus serve 后端。浏览器内处理与本地后端模式的数据路径和资源约束不同，汇报时不应混成一种部署方式。\[官方项目与启动说明\]\(https://github\.com/abhigyanpatwari/GitNexus\)



\!\[GitNexus 官方界面：仓库探索、代码、中心图与 Nexus AI 面板\]\(four\_tools\_sources/gitnexus\_ui\)



截图来源：[官方 README 引用的界面图](https://github.com/user-attachments/assets/cc5d637d-e0e5-48e6-93ff-5bcfdb929285)。



**大颗粒功能：**仓库导入与选择；文件/符号探索和搜索；关系图过滤；代码查看与引用；社区与调用流程探索；AI 对话；图查询入口。流程列表及流程图是比单个函数更粗的探索层，适合介绍系统功能路径。\[架构与 Web 说明\]\(https://github\.com/abhigyanpatwari/GitNexus/blob/d2a43e33df7d30cf17d9183beab655c0b915a1c6/ARCHITECTURE\.md\)、\[流程面板源码\]\(https://github\.com/abhigyanpatwari/GitNexus/blob/d2a43e33df7d30cf17d9183beab655c0b915a1c6/gitnexus\-web/src/components/ProcessesPanel\.tsx\)



**交互特点与局限：**图、源码与对话同时出现，适合带着问题探索仓库。浏览器模式受内存等资源约束，大仓库需要评估；社区与流程的可视化不能代替解析准确率说明。影响分析、变更分析等 MCP 能力不能自动写成已经验证的 Web 一键操作。



### **6\.3 Codebase\-Memory：3D 关系图与本地索引管理**



**入口与形态：**通过 codebase\-memory\-mcp \-\-ui=true \-\-port=9749 启用本地 UI。当前前端有 Graph、Projects、Control 三个主要标签；具体发行包是否包含 UI 需按安装渠道确认。\[项目说明\]\(https://github\.com/DeusData/codebase\-memory\-mcp\)、\[App 源码\]\(https://github\.com/DeusData/codebase\-memory\-mcp/blob/59a05eb1bf9e11deb060d782cd7d3a29f2ae2866/graph\-ui/src/App\.tsx\)



\!\[Codebase\-Memory 官方界面：3D 图、类型过滤与项目文件结构\]\(four\_tools\_sources/memory/docs/graph\-ui\-screenshot\.png\)



截图来源：[官方 graph\-ui\-screenshot\.png](https://github.com/DeusData/codebase-memory-mcp/blob/59a05eb1bf9e11deb060d782cd7d3a29f2ae2866/docs/graph-ui-screenshot.png)。



**大颗粒功能：**Graph 提供 3D 关系探索、目录与类型过滤、搜索和节点详情；详情可以查看源码及入边/出边。Projects 提供添加/索引项目、进度与规模统计、删除索引等管理。Control 提供工具进程、资源状态和日志观察；这里的进程状态不是被分析程序的运行时调用轨迹。\[节点详情\]\(https://github\.com/DeusData/codebase\-memory\-mcp/blob/59a05eb1bf9e11deb060d782cd7d3a29f2ae2866/graph\-ui/src/components/NodeDetailPanel\.tsx\)、\[项目管理\]\(https://github\.com/DeusData/codebase\-memory\-mcp/blob/59a05eb1bf9e11deb060d782cd7d3a29f2ae2866/graph\-ui/src/components/StatsTab\.tsx\)、\[控制面板\]\(https://github\.com/DeusData/codebase\-memory\-mcp/blob/59a05eb1bf9e11deb060d782cd7d3a29f2ae2866/graph\-ui/src/components/ControlTab\.tsx\)



**交互特点与局限：**比单纯图查看器多了索引管理能力，适合借鉴到工程工作台。从截图判断，密集 3D 图存在遮挡和路径追踪负担；这是界面分析判断，尚非用户实验结论。未核实内置聊天面板；与模型交互主要通过外部 MCP 客户端。



### **6\.4 Graphify：可交付的交互图和报告**



**入口与形态：**典型输出包括 graphify\-out/graph\.html、graph\.json 与 GRAPH\_REPORT\.md。用户可以打开生成的 HTML 浏览，不必将其理解为完整仓库 IDE 或持续运行的管理平台。\[项目说明\]\(https://github\.com/Graphify\-Labs/graphify\)



\!\[Graphify 官方交互图示例：社区分组与网络关系探索\]\(four\_tools\_sources/graphify\_ui\.png\)



截图来源：[Graphify 官方仓库展示图](https://github.com/Graphify-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/docs/graph-hero.png)；图片用于说明可视化风格，功能清单以当前 HTML 导出器和文档为准。



**大颗粒功能：**全图与社区浏览；节点搜索；社区筛选；节点详情与来源；邻接关系探索；图和报告导出。另有路径查询、解释、更新/监视及调用流程输出等能力，部分属于 CLI、助手入口或单独生成页面，不应全部放进主 HTML 界面功能列表。\[HTML 导出器\]\(https://github\.com/Graphify\-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/graphify/exporters/html\.py\)、\[调用流程 HTML 实现\]\(https://github\.com/Graphify\-Labs/graphify/blob/26b02b5e3430e4ab85dd7e72c7b98836d8e65c48/graphify/callflow\_html\.py\)



**交互特点与局限：**适合展示代码与文档之间的联系，也便于共享成果；当前生成图不是完整的源码联动 IDE。大图导出存在社区聚合等降规模处理路径，不能假设所有节点永远逐一显示。主 HTML 图中未核实集成通用聊天界面；网站上的未来产品计划不计入现有能力。



## **7\. 大颗粒功能对照：区分界面能力与工具能力**



标记含义：**Web**＝在查看器/应用中核实；**CLI/MCP**＝工具提供能力，但未等同认定有对应 Web 操作；**输出**＝生成独立文件或页面；**未核实**＝本次证据不足，并非断言没有。表中省略了与汇报无关的细粒度按钮。



|功能域|CodeGraph|GitNexus|Codebase\-Memory|Graphify|
|---|---|---|---|---|
|建图与更新|CLI/监视|CLI，浏览器模式另有导入处理|CLI/MCP；Web 项目索引管理|CLI/助手；缓存、更新与监视|
|工程与模块概览|Web：Map、文件与入口|Web：仓库、社区、流程|Web：Projects、图统计与目录|输出 Web：社区图与报告|
|符号搜索与源码阅读|Web：搜索、Symbol、文件源码|Web：搜索、文件树、代码面板|Web：搜索、节点详情、源码|Web：节点搜索与来源；非完整源码工作台|
|调用/关系路径探索|Web：调用邻域、Flow、轨迹|Web：图、流程列表与流程图|Web：关系邻域；CLI/MCP：追踪|Web：邻接；CLI：路径；输出：调用流程|
|变更影响与辅助分析|CLI/MCP：影响与受影响测试等|CLI/MCP：影响、变更等|CLI/MCP：调用与变更关系等|结构路径与图分析可用；专门变更工作流未核实|
|AI 问答|外部 MCP 客户端|Web AI 面板及外部 Agent|外部 MCP 客户端|外部助手/查询流程；主 HTML 非聊天应用|
|文档与非代码知识|不作为本次核实的核心|有知识组织/文档生成功能；非 Graphify 同类输入流程|有 ADR 等辅助知识入口；非多模态抽取主线|核心能力：代码、文档及多媒体语义融合|
|成果保存与导出|Web：探索轨迹、部分视图图片|图查询、文档等工具输出；具体图导出按钮未核实|持久化索引与查询结果；演示图导出未核实|graph\.json、HTML、报告及多种图/知识导出|
|状态与运行管理|索引状态与变更提示|仓库/索引及后端状态|Web：项目进度、工具进程与日志|CLI 更新/监视与生成报告|



**界面定位可以概括为：CodeGraph 适合“沿源码读关系”，GitNexus 适合“沿模块和流程问问题”，Codebase\-Memory 适合“维护和查询本地图谱”，Graphify 适合“把代码与文档知识连起来并交付图与报告”。**这是基于当前功能的产品分析，不是性能排名。



## **8\. 对你的系统：可借鉴的界面与可验证的研究增量**



### **8\.1 建议的六个功能模块**



|模块|用户要完成的任务|可借鉴对象|Rust 专项增量|
|---|---|---|---|
|工程与配置总览|选择 workspace、crate、target 与 features，查看索引是否有效|Codebase\-Memory 的项目管理、GitNexus 的仓库入口|清楚显示本次分析的 Cargo 配置与工具链|
|API 与类型浏览|搜索函数/类型，理解 trait、impl、泛型与关联类型|CodeGraph 的 Symbol 和源码联动|展示声明、绑定结果与解析不确定性|
|调用与依赖探索|从入口追调用、看跨 crate 依赖、检查变更影响|CodeGraph Flow、GitNexus Processes|区分静态目标、动态候选、外部依赖与配置有效性|
|编译诊断与 unsafe 导航|按 crate、错误类别与源码位置定位问题|各工具的筛选与节点详情|将诊断、unsafe 语法位置及分析告警分别呈现|
|源码与分析证据联动|从图上的事实回到源码及 HIR/MIR 证据|CodeGraph 的源码/关系并列|按需展开编译表示，避免默认展示全部底层细节|
|知识查询与汇报导出|提出结构或功能问题，得到可检查的答案与图|GitNexus AI、Graphify 的报告与导出|每项结论带来源和配置；推断与事实显式区分|



建议首页从工程和具体任务进入，不默认渲染全部 Rust 实体为一张大图。关系图用于解释局部问题；列表、源码、诊断和路径视图共同承担操作任务。



### **8\.2 建议的最小验证集**



1\. **配置正确性：**同一仓库至少两种 feature/target 配置，检查互斥关系是否被区分；记录所选配置，不追求穷举全部组合。

2\. **解析正确性：**覆盖同名函数、trait/impl、泛型约束、关联类型、UFCS、dyn Trait；分别测已解析边精确率、召回率和未解析比例。

3\. **宏与生成代码：**覆盖声明宏、过程宏和生成代码；检查展开实体与原位置能否关联，并记录无法分析的原因。

4\. **借用分析证据：**覆盖 move、共享/可变借用、drop、NLL 等小型样例；以相应编译阶段证据验证事实，不以自然语言回答是否流畅作为依据。

5\. **增量一致性：**分别修改普通函数、trait、宏、Cargo\.toml 和 feature 配置，检查受影响事实是否更新；同时测时间与内存。

6\. **用户任务价值：**在定位实现、解释调用、分析影响三个任务上比较纯搜索、现有图工具与本项目；记录正确率、时间、交互次数和证据可检查性。



未运行这些实验之前，汇报中应称“拟补齐的能力与预期价值”，而不是已经证明的技术优势。



## **9\. 建议的汇报顺序（12 页）**



|页|标题|本页只讲清楚的重点|
|---|---|---|
|1|目标与问题|从 Rust 工程结构走向有配置与编译证据的统一表示|
|2|四工具全景|固定项目身份、定位与比较口径|
|3|CodeGraph 与 Codebase\-Memory|同为解析/规则主导，分别侧重源码探索与持续索引|
|4|GitNexus 与 Graphify|前者增强社区/流程，后者融合代码与文档|
|5|构建方式归类|两类主路线；存储、MCP、LLM 问答是其他维度|
|6|论文证据：结构图|P1—P4 说明图如何服务检索、定位与 Agent|
|7|论文证据：语义与可靠性|P5—P8 说明多源融合、深层分析及评估边界|
|8|现有设计优点|轻量索引、源码证据、局部探索和持续维护|
|9|面向 Rust 的差距|用 cfg、trait 调用和借用三个例子解释缺口|
|10|四工具界面|四张官方截图，强调各自交互重点|
|11|功能矩阵与本项目界面|从工程、源码、路径、诊断进入；图为任务服务|
|12|研究增量与验证|说明准备实现什么、如何证明比现有基线更好|



## **附录：证据与阅读说明**



本报告引用项目官方仓库、文档、源码和论文原文。论文阅读重点为方法、图模式、实验设计与局限；没有把摘要或宣传中的效能说法作为统一测评结论。四工具源码核查不是完整代码审计，也没有证明未出现于材料中的能力不存在。



为便于复核，four\_tools\_sources 保存了仓库元数据、提交信息、文件树，以及本次读取的部分源码、文档和官方截图。完整提交号为：CodeGraph ba3c21e50d9129d2f5f3843ec3728868ae6d47a1；GitNexus d2a43e33df7d30cf17d9183beab655c0b915a1c6；Codebase\-Memory 59a05eb1bf9e11deb060d782cd7d3a29f2ae2866；Graphify 26b02b5e3430e4ab85dd7e72c7b98836d8e65c48。



论文与工具的对应关系必须保留：P1 是 Codebase\-Memory 直接论文；P2—P8 是用于比较方法与支持设计讨论的相关工作。CodeGraph、GitNexus、Graphify 未找到明确匹配论文，只报告“本次未检索到”，不推断永远不存在。



