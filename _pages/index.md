---
layout: default
permalink: /
---

# A trip in operations and development

{% for post in site.posts %}
 - {{ post.date | date: "%Y-%m-%d" }} <a href="{{ post.url }}">{{ post.title }}</a>
{% endfor %}

