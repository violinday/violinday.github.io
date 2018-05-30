---
layout: page
title: About
description: 技术改善世界，改变生活
keywords: Violinday
comments: true
menu: 关于
permalink: /about/
---

我是Violinday，一个有艺术修养的程序员 ;)

喜欢探索世界，喜欢技术

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
