---
title: mac上hexo的mathjax配置
date: 2019-11-26 11:50:08
tags:
- 配置
categories:
- 配置
---

博文中要写公式是难免的，因为配置hexo支持数学公式是必要的。 Next 主题提供了两个渲染引擎，分别是 mathjax 和 katex，后者相对前者来说渲染速度更快，而且支持更丰富的公式。我这里hexo是4.0版本了，因此又折腾了下。

###### 1，更改next下的config

配置next主题里的_config如下，只需要改一个地方就是mathjax的enable为true。

```
# Math Formulas Render Support
math:
  # Default (true) will load mathjax / katex script on demand.
  # That is it only render those page which has `mathjax: true` in Front-matter.
  # If you set it to false, it will load mathjax / katex srcipt EVERY PAGE.
  per_page: true

  # hexo-renderer-pandoc (or hexo-renderer-kramed) required for full MathJax support.
  mathjax:
    enable: true
    # See: https://mhchem.github.io/MathJax-mhchem/
    mhchem: false
```

###### 2, 去掉hexo自带的数学渲染

```
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```

在修改下源文件。打开`node_modules/hexo-renderer-kramed/lib/renderer.js`，将

```
// Change inline math rule
function formatText(text) {
    // Fit kramed's rule: $$ + \1 + $$
    return text.replace(/`\$(.*?)\$`/g, '$$$$$1$$$$');
}
```

改为：

```
// Change inline math rule
function formatText(text) {
    return text;
}
```

卸载hexo-math，安装新的。

```
npm uninstall hexo-math --save
npm install hexo-renderer-mathjax --save
```

在修改源文件，打开`node_modules/hexo-renderer-mathjax/mathjax.html`，将最后一句script改为：

```
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML"></script>
```

打开`node_modules/kramed/lib/rules/inline.js` : 

```
// escape: /^\\([\\`*{}\[\]()#$+\-.!_>])/,  注释掉改为下面一句
escape: /^\\([`*\[\]()# +\-.!_>])/,
```

下面的em渲染也改了:

```
// em: /^\b_((?:__|[\s\S])+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/, 注释掉改为下面一句
em: /^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
```



###### 3，开启bolg下的config支持

在末尾添加内容。

```
mathjax:
    enable: true
```

就可以了，鉴于之前的博客可能有些老了，配置了半天就记录下。



###### 4，最后自己在写bolg的时候头部加上mathjax: true，表示本文要数学公式渲染。



