---
title: Home
has_toc: true
has_children: true
---
# Taming Penguins

Mostly Linux, Tech, and DIY.
<hr>

### Recent Posts
<ul>
{% for collection in site.collections %}
    {% if collection.label != 'posts' %}
        {% for item in site[collection.label] limit: 3 %}
          {% if item.title != "Ansible" and item.title != "General" and item.title != "DIY" %}
            <li><strong>{{ item.collection }}:</strong> <a href="{{ item.url }}">{{ item.title }}</a></li>
          {% endif %}
        {% endfor %}
    {% endif %}
{% endfor %}
</ul>