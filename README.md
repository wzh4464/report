# Context Graph 与 Agent Memory 报告

面向 Coding Agent 的研究方向探索

## 目录结构

```
.
├── slides/                    # LaTeX 源文件
│   ├── report.tex            # 完整研究报告 (Context Graph 与 Agent Memory)
│   ├── coding_agent_dev.tex  # 开发 Coding Agent 的技术要点
│   └── ref.bib               # 参考文献
├── images/                    # 图片资源
├── notes/                     # Markdown 笔记
├── build/                     # 编译产物（gitignored）
└── README.md                  # 本文件
```

## 编译方法

### 使用 arara（推荐）

在项目根目录运行：

```bash
# 编译完整报告
cd slides && arara report.tex && cd ..

# 编译开发要点 PPT
cd slides && arara coding_agent_dev.tex && cd ..
```

### 使用 XeLaTeX 手动编译

```bash
cd slides
xelatex -output-directory=../build report.tex
biber ../build/report
xelatex -output-directory=../build report.tex
xelatex -output-directory=../build report.tex
```

## 内容概览

### report.tex - 完整研究报告
- 问题背景：从 LLM 到 Context Graph
- Context Graph 概念框架
- 相关技术（GraphRAG、Agent Memory）
- 研究方向探索（Bug 修复、漏洞检测）
- 开放问题与未来工作

### coding_agent_dev.tex - 开发技术要点
- 核心洞察总结
- 开发 Coding Agent 的 7 个技术要点
- 业界实践与趋势（主流产品、开源框架）
- 实施路线图（4 个阶段）
- 风险与应对策略

## 参考文献

详细文献列表见 `slides/ref.bib`

## License

内部研究资料
