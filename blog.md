---
layout: default
title: Posts
permalink: /blog/
---

<h1><strong><ins>Blog Posts</ins</strong></h1>
<br>

{% assign current_year = "none" %}

<ul>
{% for post in site.posts %}
  {% assign post_year = post.date | date: "%Y" %}
  {% if post_year != current_year %}
    {% if current_year != "none" %}
      </ul>
    {% endif %}
    <h2>{{ post_year }}</h2>
    <ul>
    {% assign current_year = post_year %}
  {% endif %}
  <li>
    <a href="{{ post.url }}">{{ post.post_title }}</a>
    <p>{{ post.date | date: "%B %d, %Y" }}</p>
  </li>
{% endfor %}
</ul>
