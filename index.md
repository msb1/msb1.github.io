---
layout: default
title: Home
---

# {{ site.title }}

<p class="tagline">{{ site.description }}</p>

{% assign about = site.pages | where: "name", "about.md" | first %}

## Core expertise

{% for item in about.expertise %}
- {{ item }}
{% endfor %}

## Table of contents

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
