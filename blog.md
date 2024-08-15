---
layout: default
title: Posts
permalink: /blog/
---

<h1><strong>Blog Posts ✍🏻</strong></h1>
<hr>

<ul>
{% for post in site.posts %}
  <li>
    <a href="{{ post.url }}">{{ post.post_title }}</a>
    <p>{{ post.date | date: "%B %d, %Y" }}</p>
  </li>
{% endfor %}
</ul>
