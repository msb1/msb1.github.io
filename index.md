---
layout: default
title: Home
---

# Tech Blog

Short introduction to your blog.

## Posts

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }})  
  <small>{{ post.date | date: "%B %-d, %Y" }}</small>
{% endfor %}
