---
layout: post
title: All Posts
---

{% for post in site.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <time>{{ post.date | date: "%B %d, %Y" }}</time>
  <p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
  <hr>
{% endfor %}