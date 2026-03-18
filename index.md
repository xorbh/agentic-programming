---
layout: default
title: Home
---

# Posts

<ul class="post-list">
{% for post in site.posts %}
<li>
  <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}{% if post.tags %} · {% for tag in post.tags %}<span class="tag">{{ tag }}</span>{% endfor %}{% endif %}</p>
  <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
  <p class="excerpt">{{ post.excerpt | strip_html | truncatewords: 40 }}</p>
</li>
{% endfor %}
</ul>
