---
title: Home
has_toc: true
has_children: true
---
# Taming Penguins

Mostly Linux, Tech, and DIY.

Recent posts:
<ul>
{% for post in site.posts limit: 5 %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

{: toc}