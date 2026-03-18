---
layout: default
title: Home
---

# Agentic Programming

Exploring patterns, tools, and practices for AI-assisted software development.

## Posts

{% for post in site.posts %}
- **[{{ post.title }}]({{ post.url | relative_url }})** — {{ post.date | date: "%B %d, %Y" }}

  {{ post.excerpt | strip_html | truncatewords: 30 }}

{% endfor %}
