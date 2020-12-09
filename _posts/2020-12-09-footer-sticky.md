---
layout: post
title: "css实现footer贴底"
date: 2020-12-09
excerpt: 
tags: [前端, html, css]
feature: 
comments: false
---

在写前端页面的时候，可能希望实现这样一种效果：底部有一个页脚，如果内容不够一个窗口则置于窗口底部，否则直接放到内容下面。

[网上](https://www.jianshu.com/p/262dac139e2e)找了个教程，没太折腾明白，拼拼凑凑效果也一般，看了一点css布局以后有了点理解，记录一下。

假设有一个footer:

```html
<footer class="footer">
  <div class="container">
    ...
  </div>
</footer>
```

希望这个footer贴底需要用如下的样式：

```css
html {
  /* 相对定位布局的页面，最小高度100%表示占满整个高度 */
  position: relative;
  min-height: 100%;
}

body {
  /* 需要在内容下面留出足够放下footer的间距，否则会产生重叠 */
  margin-bottom: 60px;
}

.footer {
  /* footer使用绝对布局，相当于抽离出了另一个图层 */
  position: absolute;
  /* 底边距0，贴底 */
  bottom: 0;
  width: 100%;
  /* 设置高度 */
  height: 50px;
  background-color: #f5f5f5;
}
```

