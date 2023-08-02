# LaTeX基础概念

- 命令以反斜线`\`开头

- `\documentclass[options]{class-name}` 必须出现在每个LaTeX文档的开头
  
  - `options`：选项，指定一些排版参数
  - `class-name`：文档类型，有如下的文档类型

- `\usepackage[options]{package-name, package-name, ...}`：指定宏包

- `%`作为注释

- 特殊字符用`\`形式输入

- `\`本身需要用`\textbackslash`

- `\\`表示手动换行

## 文档类型

- article：文章格式的文档类，用于科技论文、报告、说明文档等
- book：书籍文档类，包含章节结构、前言、正文、后记等结构
- report：长篇报告格式的文档类，具有章节结构，用于综述、长篇论文、简单的书籍等
- proc：基于article文档类的一个简单的学术文档模板
- slides：幻灯片格式的文档类，使用无衬线字体
- minimal：一个精简的文档类，只设定了纸张大小和基本字号，用作代码测试的最小工作示例
- moderncv
- beamer
- ctexart：支持中文排版的article
- ctexbook：支持中文排版的book
- ctexrep：支持中文排版的report
- ctexbeamer：支持中文排版的beamer

## options选项

- 字体大小：默认为 `9pt`
- 纸张大小：
  - `a3paper`
  - `letterpaper` （默认）
  - `a4paper`
  - `b4paper`
  - `executivepaper`
  - `legalpaper`
- 单面、双面排版：
  - `twoside`：双面，book默认为双面
  - `oneside`：单面，article和report默认为单面
- 单栏、双栏排版：
  - `onecolumn`：单栏，默认为单栏
  - `twocolumn`：双栏
- 指定新的一章`\chapter`是在奇数页（右侧）开始还是直接紧跟着上一页开始
  - `openright`：book默认为`openright`
  - `openany`：report默认为`openany`
- 横向、纵向排版
  - `landscape`：横向排版，默认为纵向
- 指定标题命令`\maketitle`是否生成单独的标题页
  - `titlepage`：report和book默认为`titlepage`
  - `notitlepage`：article默认是`notitlepage`
- `fleqn`：行间公式左对齐，默认为居中对齐
- `leqno`：公式编号放在左边，默认在右边
- `draft`、`final`指定草稿、终稿模式，默认为`final`