---
layout: archive
title: Blog
author_profile: true
permalink: /blog/
---

{% for post in site.posts %}
  {% include archive-single.html type="list" %}
{% endfor %}
