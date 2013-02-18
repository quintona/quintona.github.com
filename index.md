---
layout: page
title: Ramblings of a software Engineer
tagline: The only thing that makes life possible is permanent, intolerable uncertainty; not knowing what comes next.
---

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



