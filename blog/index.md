---
layout: default
title: Publications
---

# {{ page.title }}

<ul class="posts">
  {% for post in site.posts %}
  <li>
    <span>{{ post.date | date_to_string }}</span> : <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a><br>
    {% if post.desc %}
      <p style="color:#808080; font-style:italic">{{ post.desc }}</p>
    {% endif %}
  </li>
  {% endfor %}
</ul>