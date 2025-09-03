---
layout: page
title: All posts
permalink: /posts/
---

<ul class="post-list">
{% for post in site.posts %}
  <li style="margin-bottom:1.25rem">
    <h3 style="margin:0 0 .2rem">
      <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h3>
    <span class="post-meta">
      {{ post.date | date: "%b %-d, %Y" }}
      · {%- assign words = post.content | number_of_words -%}{%- assign mins = words | divided_by:200 | plus:1 -%} {{ mins }} min read
    </span>
    {% if post.excerpt %}
      <p class="post-excerpt">{{ post.excerpt | strip_html | strip_newlines | truncate: 220 }}</p>
    {% endif %}
  </li>
{% endfor %}
</ul>
