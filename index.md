---
layout: page
title: Landing ...
---
{% include JB/setup %}

I need a playground for writing blog posts and here it is. Please have a look at [Kreuzwerker blog](https://kreuzwerker.de/blog) also. 


Recent posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



