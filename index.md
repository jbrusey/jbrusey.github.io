---
layout: single
title: Home
author_profile: true
classes: wide
---

# James Brusey

Professor of Computer Science.

I work on artificial intelligence, cyber-physical systems, machine learning, sensing, optimisation, and applied data science.

## Sections

- [Blog](/bloglist/)
- [Projects](/projects/)
- [Publications](/publications/)
- [Teaching](/teaching/)
- [About](/about/)

## Recent posts

{% for post in site.posts limit:5 %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%-d %B %Y" }}
{% else %}
- No posts yet.
{% endfor %}

[View all posts](/bloglist/)
