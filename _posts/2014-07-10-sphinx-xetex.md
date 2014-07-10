---
layout: post
title: 使用 XeTeX 为 Sphnix 输出 PDF 格式的文档
date: 2014-07-10 15:33:36
tags: sphinx latex
---
Sphinx 生成 PDF 输出时，使用的pdflatex + inputenc，当文档出现 UTF8 字符时，make latexpdf 就会出错，如：

    ! Package inputenc Error: Unicode char \u8:工 not set up for use with LaTeX.

解决的办法是使用 XeTeX 替换 pdflatex：

* 从 TeX 源文件中删去 inputenc 相关的配置

        \usepackage[utf8]{inputenc}
        \DeclareUnicodeCharacter{00A0}{\nobreakspace}

* 在 TeX 源文件的 `\title` 之前加入 XeTeX 相关的配置

        \usepackage{xeCJK}
        \setCJKmainfont{WenQuanYi Micro Hei}
        \setCJKmonofont{WenQuanYi Micro Hei Mono}
    
    后面2行中的字体名称根据环境中实际安装的字体进行调整。

* 最后，使用 xelatex 命令生成 PDF 文件

        # xelatex *.tex

上述步骤中对 TeX 源文件的修改也可以通过修改配置在 sphnix 生成 TeX 源文件时完成。

修改 conf.py 中 latex_elements 配置项，如：

    latex_elements = { 
        'inputenc': '', 

        'preamble': ''' 
            \usepackage{xeCJK}
            \setCJKmainfont{WenQuanYi Micro Hei}
            \setCJKmonofont{WenQuanYi Micro Hei Mono}
            ''',
    }

不过使用上述配置时，生成的 TeX 源文件中

    \DeclareUnicodeCharacter{00A0}{\nobreakspace}
会保留，导致 xelatex 执行失败，仍然需要手动删除。
