# 论文自动排版 Skill 技术草案（基于 ZUFE-Thesis）

## 1. 目标

构建一个可复用的 Codex Skill：
- 输入：用户论文元信息 + 章节内容 + 参考文献 + 附录/代码（可选）
- 输出：符合 ZUFE 模板的可编译 LaTeX 项目（最小人工修订）
- 质量目标：一次编译成功率 > 90%，格式违规项可自动提示与修复

## 2. 与现有项目的映射关系

- `main.tex`：文档入口与装配流程，Skill 仅应写入被允许用户修改的区域，不触碰类文件。
- `chapters/basicinfo.tex`：元信息配置中心（题目、作者、学院、摘要、关键词、reportStyle）。
- `chapters/mainbody.tex`：章节拼装入口。
- `misc/cover.tex`、`misc/abstract.tex`、`misc/reference.tex`：封面/摘要/参考文献版式由模板固定，Skill 主要供数值和文本。
- `Reference.bib`：参考文献数据库，Skill 负责归一化 BibTeX。
- `zufe.cls`：样式内核（页边距、字号、行距、标题、目录、图表、算法、代码样式等），默认只读。

## 3. Skill 架构建议

### 3.1 目录结构（建议）

```text
skills/zufe-thesis-auto-typeset/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── scripts/
│   ├── scaffold_project.py
│   ├── transform_source_to_tex.py
│   ├── normalize_bib.py
│   ├── lint_tex_content.py
│   ├── compile_check.sh
│   └── post_compile_audit.py
├── references/
│   ├── zufe-template-map.md
│   ├── bib-rules.md
│   ├── section-style-rules.md
│   └── troubleshooting.md
└── assets/
    └── prompt-templates/
        ├── chapter_prompt.txt
        └── abstract_prompt.txt
```

### 3.2 流程总线

1. **输入采集**：题目、姓名、学院、章节提纲、文稿来源（Word/Markdown/纯文本）。
2. **结构标准化**：把文稿切分为引言/章节/结论/附录，映射到 `chapters/*.tex`。
3. **引用标准化**：抽取引用、补齐 BibTeX、写入 `Reference.bib`。
4. **版式注入**：填充 `chapters/basicinfo.tex` 与章节文件。
5. **编译验证**：`xelatex -> biber -> xelatex -> xelatex`。
6. **质量审计**：检查蓝字占位、未解析引用、浮动体溢出、标题层级异常。
7. **反馈回路**：生成“修订建议 + 可自动修复补丁”。

## 4. 模块逻辑详解

### 4.1 内容入口模块（transform_source_to_tex）

- 支持 Markdown/纯文本直接转换。
- Word 建议中间格式：`docx -> markdown(pandoc) -> tex`，避免直接解析 OOXML 复杂性。
- 逻辑：
  - 识别一级标题映射为 `\chapter{}`，二级 `\section{}`，三级 `\subsection{}`。
  - 将图片标记转换为 `figure` 环境并落盘到 `Images/`。
  - 将表格标记转换为 `table` + `tabular`。

### 4.2 元数据注入模块（scaffold_project）

- 对 `chapters/basicinfo.tex` 进行模板占位替换：
  - `\thesisTitle`、`\thesisTitleEN`、`\yourName` 等。
  - 根据任务类型设定 `\reportStyle`（0/1/2）。
- 若用户无副标题，则注释 `\haveSub{}` 并清空副标题字段。

### 4.3 参考文献模块（normalize_bib）

- 输入：DOI、URL、手工条目混合。
- 处理：
  - 去重（标题+年份+第一作者近似匹配）。
  - 字段补全（author/title/year/journal/booktitle）。
  - 生成稳定 citekey（`AuthorYearShortTitle`）。
- 输出：标准化 `Reference.bib`。

### 4.4 规则检查模块（lint_tex_content）

核心检查项：
- 蓝字占位是否残留（模板提醒文字必须删）。
- `\cite{}` 是否都能在 `Reference.bib` 中解析。
- 标题是否跳级（chapter 后直接 subsection）。
- 图表标题是否缺失。
- 章节中是否存在明显未替换占位（如 `xx`、`xxxx`）。

### 4.5 编译与诊断模块（compile_check + post_compile_audit）

- 编译脚本固定顺序：
  - `xelatex -interaction=nonstopmode main.tex`
  - `biber main`
  - `xelatex -interaction=nonstopmode main.tex`
  - `xelatex -interaction=nonstopmode main.tex`
- 解析日志：
  - Undefined citations
  - Overfull/Underfull boxes
  - Missing glyph（生僻字）
  - Missing file（图片/章节路径）

## 5. 基于当前仓库的关键技术判断

1. **高可行性**：项目已将用户可修改区与样式内核分离，天然适合自动化写入。
2. **中等难度**：难点在“内容语义到 LaTeX 结构”的稳定映射，而不是排版本身。
3. **高收益点**：参考文献标准化与编译诊断自动化，能显著减少手工排错时间。

## 6. 开发里程碑

### M1（1-2 周）最小可用
- 生成 `basicinfo.tex` 与章节骨架。
- 支持 Markdown 转章节 tex。
- 支持一次完整编译流水。

### M2（2-3 周）质量增强
- Bib 自动归一化。
- 规则检查器 + 自动修复建议。
- 编译日志结构化诊断报告。

### M3（2 周）体验增强
- 支持“增量更新章节”。
- 支持“只修格式不改内容”模式。
- 输出“最终提交前检查清单”。

## 7. 技术栈建议

- Python 3.11+
- `pydantic`（输入模型）
- `jinja2`（文本模板填充）
- `bibtexparser`（bib 解析）
- `panflute`/`pandoc`（文档转换）
- Shell + TeXLive + Biber（编译链）

## 8. 风险与缓解

- **风险1：Word 文稿结构不规范导致错误映射**
  - 缓解：先转 Markdown，再走统一解析。
- **风险2：生僻字字体缺失**
  - 缓解：自动提示使用 `\mystsong{}`/`\mystkaiti{}` 包裹。
- **风险3：引用不完整**
  - 缓解：将缺字段文献列入“人工补录队列”。

## 9. 预估难度

- **总体：中等偏上（6.5/10）**
- 原因：
  - 模板稳定、规则明确（降低难度）
  - 输入异构（Word/Markdown/手写）与语义转换（增加难度）

## 10. 建议的落地策略

先实现“**格式自动化**”再实现“**内容智能化**”：
1. 第一阶段只做结构填充 + 编译诊断。
2. 第二阶段再做内容重写、图表生成、术语统一。

这样可快速产出可用工具，并控制模型幻觉或误改正文的风险。
