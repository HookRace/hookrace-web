---
layout: page
title: Posts
---

{% for post in site.posts %}
  * {{ post.date | date: "%Y-%m-%d" }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endfor %}

# Tags
{% assign tags = site.tags | sort %}
{% for tag in tags %} [{{ tag[0] }}](/blog/{{ tag[0] | downcase }}/){% if forloop.last == false %} &middot; {% endif %}{% endfor %}
