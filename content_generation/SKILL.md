---
name: content_generation
description: 基于代码仓库、笔记、实验数据或论文要求，全自动智能撰写学术论文初稿的主线管线。强制分章逐批检索代码、分步输出，内置规避上下文超限机制和人工审核卡点，无缝衔接格式化引擎。
---

# 论文内容智能生成工作流 (Automated Academic Content Generation Pipeline)

本 Skill 专用于 **Pipeline D**。解决"有项目、有数据但没时间写规范八股文"的痛点，将源码或离散数据提炼串联成结构完整的学术论文初稿，严格控制幻觉和上下文崩溃。

## 0. 流程总览与组件定义

物料存放在工作区，包括：
- `resources/samples/`：要求文件或参考的优秀范例文献（PDF/DOCX/MD），非必须但如有则优先。
- `resources/outline.md`：大纲定义文件（在第一阶段由系统生成，供用户自由提件与修改）。
- `<用户代码/数据目录>`：论文本体依托的大型项目/数据集目录及其各类脚本日志。

本工程严格采用 **延迟加载 (Lazy Loading)** 和 **按需检索 (Just-in-Time Context)** 设计哲学，避免大型项目全量读取导致的上下文溢出（Token Limit Exceeded）。

## 1. 结构定调与大纲生成 (Structure Inference & Approval)

### 1.1 要求与范例解析
当用户提供了会议特定的“要求要求说明（Call for Papers）”或“参考样例论文”时，首先通过局部精读提取该会议或期刊习惯的八股文结构：
- 分析范例的章节编排、次级标题命名偏好、图表频次及典型引用，反推并套用撰写范式。

### 1.2 起草 `outline.md` 
综合用户的初步想法、工程的根目录 `README.md` 与参考范例，生成一个名为 `resources/outline.md` 的初拟大纲文件，明确细分章节结构。内容包括：
- **核心论点框架 (Key Contributions)**
- **各级章节标题 (Heading Tree)**
- **各个章节指派的检索策略 (Material Mapping)**（例如：*章节4.1需定位模型的训练函数*；*章节5需查阅 evaluation_results.csv*）

### 1.3 用户审查卡点 (Mandatory User Review)
生成大纲后，**必须**调用 `notify_user` 强阻断大模型执行流：
> "大纲已生成至 `resources/outline.md`，请您审阅。您可以直接打开该 Markdown 文件进行修改、增删或留下 TODO 备注。当您认为大纲框架完美后，请发送**继续**以启动正文生成流程。"

## 2. 超大项目上下文控制机制 (Handling Extremely Large Repositories)

用户的项目（如大型深度学习框架、仿真环境系统工程）极易突破大模型的原生长下文极限，严禁执行“递归读取整个项目所有代码”。必须采用如下策略：

1. **地图与索引制备 (Directory Tree Indexing)**：
   先使用命令行树形扫表（`list_dir` / `tree`）获取根目录至 `src/` 深处的层级。在模型心智中建立一张“文件寻址地图”，不用实际读取文件内容。
2. **章节局部视界按需挂载 (Section-Scoped Fetching)**：
   每一次推进到一个具体章节时，只聚焦取用预先大纲划定的高度相关模块。
   - *写 Related Work*：只看根文档提供的关键字，进行外部搜索拓展。
   - *写 Methodology*：针对性地执行 `grep_search` 或检索特定算法核心类，只调用 `view_file` 查阅涉及第一作者贡献的核心代码段。对于次要或冗长的系统流转代码一概屏蔽。
   - *写 Evaluation / Results*：只读取带有 log, test, result 等前缀的小型表格、JSON 或 `.csv` 片段，绝大部分无关源码可以强行从工作区卸载。

## 3. 严密的逐章节自回归 (Section-by-Section Drafting)

> [!CAUTION]
> **切勿在单次Prompt响应中生成超过1个章节的完整内容**。每次必须通过推进边界，释放和丢弃上一章拉取的细枝末节源码，以腾出清爽的上下文内存。

1. **执行边界管理**：读取 `checkpoint.json`，当前处理到 `current_unit`（依照 `outline.md` 推进）。
2. **片段素材检索**：依照 `outline.md` 对本节预判的素材方向，执行步骤 2 的地图寻址与片段阅读。如果强行查验依旧抓不到合理的验证逻辑，必须在当前节输出 `[TODO: Insert missing missing code logic/value here]` 强烈高亮留白，**硬性拒绝无端脑补**。
3. **分章输出**：用最平实考究的学术外文撰写为 `resources/md/section_{current_unit}.md`。
4. **内存悬挂冷却 (Context Flush & Suspend)**：遵循 4 节一休眠并刷新的法则（详见通用文档规范）。每一次挂起或跨章节，大模型将自动丢失旧的深层代码文件上下文，依靠精练的 `outline` 继续下一步撰写。

## 4. 终稿自检与要求对齐 (Quality Assurance & Requirements Check)

在所有分章生成完毕后，不要急于交由自动排版，**必须**执行一轮基于 Markdown 原文的全局 QA 校对。

1. **组合与统计评测**：将所有 `section_N.md` 逻辑拼装阅读，精确统计初稿实体字数 (Word Count)、独立公式数量或参考文献规模等硬性结构指标。
2. **要求严格复核**：重新读取 `resources/samples/` 中的“强制要求/投稿须知（Call for Papers）”（如“总长度不得超过 8 页，纯文本必须限制在 4000 词内”、“必须包含讨论局限性的独立章节”），以及用户在大纲环节口头追加的硬性诉求。
3. **出具自查清单 (Self-Check Checklist)**：
   - [ ] **物理约束**：字数、引用规模是否达标？
   - [ ] **意图对齐**：是否切实响应了用户最初的特色请求（如“重点强调低功耗”）？
   - [ ] **幻觉与占位**：对全文进行正则扫描，是否有任何强行留下来的 `[TODO]` 面包屑等待补全数据。
4. **拦截或返工卡点**：如果自检环节发现系统级偏离（如字数严重不足），大模型禁止“知情不报”并掩耳盗铃，必须触发打回拦截：
   > “检测到当前生成的 Markdown 底稿约为 2100 词，未达到您要求的 4000 词指标。您希望我回到 `Methodology` 或 `Experiments` 章节深度挖掘更多仓库源码进行扩写，还是强制跳过该验证？”
   发现遗留的 `[TODO]` 时，也必须合并成一份待填写的表单，提醒人工增补。

## 5. 与排版管道衔接组装 (Handoff to Pipeline C)

1. 确认上述 QA 校对全线亮绿灯，或者用户人工赦免所有告警后：构建并合并出最终版的 `resources/compiled_paper.md`。
2. 更改配置中心字典：修改 `config.json` 的 pipeline 标记为 `"C"`（表示 MD 已就绪）。
3. 闭环交付宣言：“全文字数校验与防幻觉审核完毕，Markdown 原貌已锁死。即将唤醒主管线的 **Pipeline C（MD 直转排版管道）**，为您按照指定模板重构标准 Word 版式交付！”
