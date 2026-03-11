---
layout: default
title : Blogs
---
<ul class="post-list">
  {% for post in site.posts %}
    {% if post.blog == true and post.published == true %}
    <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
    {% endif %}
  {% endfor %}
</ul>
