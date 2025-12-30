---
layout: home
title: Easter的技术博客
---

欢迎来到我的技术博客 👋  

## 最新文章
{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
