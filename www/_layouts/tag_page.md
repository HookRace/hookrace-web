---
layout: default
---
<link rel="alternate" type="application/atom+xml" title="{{ page.tag }} Posts &middot; Blog Feed" href="{{ site.baseurl }}blog/feed/{{ page.tag | downcase }}/">

<h1>{{ page.tag }} Posts</h1>

<ul>
{% for post in page.posts %}
  <li><p>{{ post.date | date: "%Y-%m-%d" }} &raquo; <a href="{{ post.url }}">{{ post.title }}</a></p></li>
{% endfor %}
</ul>
