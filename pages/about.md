---
layout: page
title: About
description: Keep Moving
keywords: lutcraft
comments: true
menu: 关于
permalink: /about/
---

You are about to embark on a long, long journey, my friend,

a journey that stretches far into the distance.

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'lutcraft.github.io/' %}
{% endif %}
</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
