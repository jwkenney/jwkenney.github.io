---
title: Home
has_toc: true
has_children: true
---
# Taming Penguins

Mostly Linux, Tech, and DIY.



{: toc}

Recent Posts:
<ul>
{% for collection in site.collections %}
    {% if collection.label != 'posts' %}
        {% for item in site[collection.label] limit: 3 %}
            <li><strong>{{ item.collection }}:</strong> <a href="{{ item.url }}">{{ item.title }}</a></li>
        {% endfor %}
    {% endif %}
{% endfor %}
</ul>