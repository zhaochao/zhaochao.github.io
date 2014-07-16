---
layout: post
title: Tip：将TeX绘制的图像输出成图片文件
date: '2014-07-16 16:41:57'
tags: tex standalone
---
画图并是不每个人都擅长的事情，很多时候更希望关注于内容，而不是展现的形式，所以连画图都希望使用TeX解决。

主要的参考在这里：

<http://tex.stackexchange.com/questions/51757/how-can-i-use-tikz-to-make-standalone-svg-graphics>

大概说明一下：

* 使用 `standalone` 包，加上 `convert` 选项，在生成pdf时自动完成 pdf --> png、svg 等的转换；
* 转换为 svg 时，使用 `convert=pdf2svg` ，系统要安装 `pdf2svg` 软件包，一张图片的话（即 TeX 文档只有一页）再加上 `multi=false` ；
* 使用 pdflatex （ xelatex ）时，加上 `-shell-escape` 参数。

举个例子：

test.tex:

    \documentclass[tikz,convert=pdf2svg,multi=false]{standalone}
    %\usetikzlibrary{...}% tikz package already loaded by 'tikz' option
    \begin{document}
    \begin{tikzpicture}% Example:
      \draw (0,0) -- (10,10); % ...
          \draw (10,0) -- (0,10); % ...
                \node at (5,5) {Lorem ipsum at domine standalonus};
    \end{tikzpicture}
    \end{document}

生成 pdf 和 svg：

    $ xelatex -shell-escape test.tex
