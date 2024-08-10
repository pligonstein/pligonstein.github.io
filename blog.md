---
layout: default
title: "Posts"
permalink: /blog/
---

<h1>Blog Posts</h1>

<ul>
{% for post in site.posts %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <p>{{ post.date | date: "%B %d, %Y" }}</p>
    <p>{{ post.excerpt }}</p>
  </li>
{% endfor %}
</ul>
