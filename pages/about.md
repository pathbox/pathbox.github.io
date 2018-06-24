---
layout: page
title: About
description:
keywords: Pathbox
comments: true
menu: 关于
permalink: /about/
---

Rubist、Gopher

Love Coding、reading、cooking and fitness

Interested in high performance high concurrence high available and distributed systems

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
