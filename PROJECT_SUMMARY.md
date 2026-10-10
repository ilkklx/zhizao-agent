# 项目总结 · 智造实演（zhizao-agent）

> 本文档是仓库的**自包含项目说明**：目标让任何读者（或 AI）只读这一篇，就能完整了解这个智能体是什么、怎么搭建、包里有什么。关键结论标注来源文件。
> 撰写时间：2026-10-10。基于对仓库全部文件的通读核对（含工作流 JSON 的结构化解析）。

---

## 1. 一句话定位

"智造实演"是一个**面向中学信息科技/通用技术课堂的 AGV 仓储小车避障实验教学智能体**：学生用自然语言指挥小车，平台经意图识别 + 参数提取调用工作流，由 Python 代码节点运行 **A\* 算法**规划避障路径，最后以交互卡片（路径动画 + 折线图）返回结果。参赛赛事：2026 年广东省"元智新潮"大学生智能体应用创新大赛。

## 2. 平台与包结构

- 来源 `原始包/智造实演_v20260908092446_package/metadata.json`：`PackageType=ApplicationPackage`、`PackageVersion=1.0.0`、`PackageId=2103160071964912064`、导出时间 2026-09-25、含组件哈希（App=CC38AF502963E404、KB=ABFA5699AF071C61）；平台为**腾讯云智能体平台**（模型别名 Youtu/*、Hunyuan/* 可证）。
- 三层结构：外层 zip（metadata + app.zip + knowledge_bases.zip）；`app.zip` → `app/`（agent / app_config / kb_config / workflow）；`knowledge_bases.zip` → `kb_2095864559561684864.zip`（知识库本体）。
- 仓库同时保留**原始 zip**与**解包目录**（方便回导或直接阅读）。

## 3. 智能体设计（app/agent/agent_config.json）

- AgentId：abb75da0-e79e-4ca7-9c77-f1f32d435921；
- **路由指令原文**（人设核心）："如果用户询问传感器原理、AGV 安全规则、算法概念、课程知识等学习类问题，请直接使用知识库问答回答，不要调用工作流；只有用户明确要求规划路径……才调用工作流"；HandoffDescription："AGV 仓储小车避障实验教学助手"；
- **模型配置**：DeepSeek-V4-Flash，Temperature 0、TopP 0.5、MaxTokens 1000；MaxReasoningRound 15；保留历史 50 条；开启澄清反问；
- **工具**：仅知识库问答插件（youtu-mrc-pro，KnowledgeScope=AllKnowledge）；
- 问候语（base_config.json）："欢迎来到《智造实演》AGV 避障实验！……点击方向按钮自由指挥小车"；
- 应用变量：car_path、car_position（初值 [[0,2]] / [0,2]）；同义词表为空。

## 4. 工作流（app/workflow/）

名称"计算小车避障路径并返回折线图数据"（导出名带"-副本3"后缀）。**31 节点、42 条边**，节点类型统计：LOGIC_EVALUATOR ×10、WIDGET ×7、PARAMETER_EXTRACTOR ×4、ANSWER ×3、CODE_EXECUTOR ×3（均 Python3）、START/END/INTENT_RECOGNITION/LLM_KNOWLEDGE_QA 各 ×1。

主链路：`开始 → 参数提取1（action/direction/step；约束原文"输入'往右走'→{action:run,direction:right,step:0}"）→ 条件判断 if-else 阶梯（task/map/sensor/params/run/next/prev/reset/qa/else）→ 任务书 → 选传感器 → 参数提取4（无法判断时默认 ultrasonic）→ 地图搭建 → 参数调试 → 参数提取2 → 代码1（A\*+heapq，约 7.7KB）→ 代码2（动画渲染，约 6.2KB）→ 动画卡片 → 结束`；`qa 分支` → 意图识别（"同时包含路径参数和知识问题时，优先识别为路径规划"）→ 知识问答或引导回复。

- **三传感器膨胀半径**（代码节点定义）：红外 0.5（半径 0）、超声波 2.0（半径 1）、激光雷达 3.0（半径 2）；
- **交互卡片**：任务书、参数调试（默认 10×10 地图、障碍 [[2,1],[3,0],[5,5]]）、选传感器、地图搭建、路径动画、折线图（演示 (0,0)→(4,4) 共 8 步）、失败提示。

## 5. 知识库（knowledge_bases/kb_2095864559561684864/）

KB 名称"智造实演"，CharSize 37 284。4 篇 Markdown（含全文与切块）：

| 文档 | 内容 |
|---|---|
| AGV安全规范要点.md | 依 GB/T 45750-2025，含 A–E 防护等级表 |
| 传感器知识.md | 三类传感器原理对比及膨胀半径映射 |
| 课标要点_算法与人工智能.md | 义教 2022 + 高中 2017 课标 |
| 教材算法与AI要点.md | 人教中图版/粤教版，A\* 启发式搜索要点 |

切块：segment.jsonl 各 15/7/10/7 条（字段 SegmentId/Title/PageContent/BigDataId 等）；big_data.jsonl 每篇一行全文；category 两类均"未分类"；qa/、schema/ 共 6 个 jsonl 均为 0 字节（平台占位文件）。挂接配置（app/kb_config/kb_refs.json）：QA 置信 0.7 / TopN 3、Doc 置信 0.2 / TopN 5、Rerank 开启、检索策略 Mixing。

## 6. 技术栈一览

| 项 | 说明 |
|---|---|
| 平台 | 腾讯云智能体平台（应用包导出格式） |
| 主模型 | DeepSeek-V4-Flash（agent_config.json） |
| 平台组件 | youtu-intent-pro / youtu-mrc-pro / youtu-embedding / youtu-reranker / youtu-rewrite / youtu-parsing / hunyuan-turbos-vision（base_config.json）；长期记忆 DeepSeek-V3-0324 已禁用 |
| 算法 | A\* 启发式搜索 + heapq（工作流代码节点） |
| 前端 | Widget 交互卡片（HTML） |

## 7. 复现方式

包为平台标准导出格式（含组件哈希），设计上可回导平台重新部署；**回导操作未在本机实测**，平台功能入口以官方文档为准。直接阅读/改造成其他平台，可从 `智造实演-智能体设计说明书.md` + `app/workflow/workflow_*.json` 入手（提示词、参数约束、代码节点均为明文）。

## 8. 问题与异常清单（通读发现）

1. **名称不统一**：app 侧显示名仍为"我的智能体应用16"（平台默认名），知识库侧已改名"智造实演"；工作流导出名带"-副本3"后缀——均为平台导出时的原始状态，未修改；
2. **设计说明书与实际配置的差异（存疑）**：说明书提到主智能体模型"gpt-5.6-sol"，而实际 agent_config.json 为 DeepSeek-V4-Flash；参数提取1 节点模型别名显示"OpenAI/TZDlPCJE"（平台内部别名，存疑）——如需对外答辩引用，建议另行核对；
3. **平台元数据**：metadata.json 记录平台导出人显示名（"微信用户"开头的平台昵称）、base_config.json 头像为平台资源 URL（带平台服务器 IP）——属平台导出原样内容，非额外个人信息；
4. 知识库 4 篇文档引用的配套"全文提取"txt 文件不随包（asset/document 内仅 4 个 md，含全文）；
5. 未发现损坏文件；未发现真实密钥。

## 9. 文件导览

- `原始包/`：平台原始导出三段式（metadata.json / app.zip / knowledge_bases.zip）；
- `app/app/agent/agent_config.json`：人设与路由指令；`app/app/agent/asset/image/`：AGV 小车白色插画图标；
- `app/app/app_config/`：base_config（模型与问候语）、app_variable（car_path/car_position）、app_synonyms；
- `app/app/kb_config/kb_refs.json`：知识库挂接参数；
- `app/app/workflow/workflow_cf8af0e2-*.json`（约 209KB）：完整节点图；`workflow_config.json`（约 17KB）：工作流配置；
- `knowledge_bases/`：kb zip 原件 + 解包目录（4 篇 md + 切块数据）；
- `智造实演-智能体设计说明书.md`：设计说明书（含匿名合规声明，署名"智造实演团队"）。
