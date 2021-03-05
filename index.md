---
title: Home
has_toc: true
has_children: true
---
# Taming Penguins

Mostly Linux, Tech, and DIY.

Latest posts via collections:
<ul>
{% for collection in site.collections %}
    {% if collection.label != 'posts' %}
        {% for item in site[collection.label] limit: 1 %}
            <li><strong>{{ site[collection.label] }}:</strong> <a href="{{ item.url }}">{{ item.title }}</a></li>
        {% endfor %}
    {% endif %}
{% endfor %}
</ul>

Last 10 posts:
<ul>
{% for post in (site.ansible site.general site_.diy) | sort limit: 10 %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

{: toc}