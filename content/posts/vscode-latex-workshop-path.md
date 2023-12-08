---
title: "LaTeX Workshop 插件的一些坑"
date: 2023-12-08T18:41:41+08:00
description: "\\relax"
categories:
  - "vscode"
  - "LaTeX"
---

记录一下为 $\LaTeX$ 文档添加 [minted](https://ctan.org/pkg/minted) 支持时使用 vscode 的插件 LaTeX Workshop 遇到的一些坑。

## TL;DR

需要注意以下几点

* minted 需要执行外部命令，即编译时需要 `-shell-escape` 参数，解决方案是在 `latex-workshop.latex.tools` 对应的 tool 的命令行参数加上该项（latexmk 或 xelatex 都行，取决于直接用 latexmk 编译还是走 xe->bib->xe->xe 的 recipe）
* minted 需要 [Pygments](https://pygments.org/)，确保在某个 python 环境中安装了该包并且 `pygmentize` 位于 Path （可通过执行 `pygmentize -V` 验证）
* LaTeX Workshop 只会读取 Path 的内容，**并不会对类型为 expandable string（即 REG_EXPAND_SZ）的值进行展开！**
* 鉴于上一点，使用时请确保 xelatex 和 pygmentize 均位于 Path，且路径均为绝对路径，不含需要展开的其他注册表变量

## 发现问题的过程

MiKTeX 安装 minted 后，使用 pipx 安装 Pygments，命令行运行 `latexmk -xelatex -shell-escape main` 可正常编译，但使用 LaTeX Workshop 的 Recipe 则报错找不到 pygmentize。使用 `\write18{python -VV}` 确定 Python 本身都不在插件读到的 Path 里，基本确定是 expandable string 没有展开的锅。

为什么命令行能直接编译？因为之前就出现过通过不同的方式打开 Windows Terminal 以后 expandable string 时而展开时而不展开的情况。解决方案是在 profile 开头加上这一句：

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

强制进行一次解析。随后通过修改 Path 直接加入 pygmentize 的路径，成功通过 recipe 编译，验证了上述分析过程。

因为我通过 expandable string 管理逐渐复杂的 Path，因此将路径不加归类直接放入顶层的 Path 必然不是最终的解决方案。最后的解决方案是在 latexmk-xe 的 recipe 里重写 Path 环境变量

```json
{
  "name": "latexmk-xe",
  "command": "latexmk",
  "args": [
      "-shell-escape",
      "-xelatex",
      "-time",
      "-interaction=nonstopmode",
      "%DOCFILE%"
  ],
  "env": {"PATH": ""}
}
```

Path 只需要包含 MiKTeX 的 bin 目录，perl 的 bin 目录（latexmk 依赖 perl）和 pipx 的 bin 目录即可（Pygments 通过 pipx 安装）。如此修改后即可正常编译导入了 minted 包的 $\LaTeX$ 文档

## 题外话

我流 Path 管理方式，以 Python 举例。先在用户环境变量里建立一个类型为 expandable string 的变量 `PYTHON_PATH`，然后将 `%PYTHON_PATH%` 加入 Path，最后修改 `PYTHON_PATH` 的内容为

```plain
%LOCALAPPDATA%\Programs\Python\Python311;%LOCALAPPDATA%\Programs\Python\Python311\Scripts;%USERPROFILE%\.local\bin
```

优点是可以给路径归类，放到不同的变量下面，然后 Path 统一引用变量，展开后为正确路径。缺点是遇到只读注册表不展开的程序就翻车了。结论上来看该方案带来的便利还是远大于缺点的，瑕不掩瑜。
