---
layout: archive
permalink: /blog/
title: "Blog Articles"
date: 2016-10-24T21:40:00-00:00
modified: 2016-10-24T21:40:00-00:00
excerpt: "The home of all of our musings"
author_profile: true
header:
  overlay_image: homesplash.jpeg
  overlay_filter: rgba(50, 50, 50, 0.5)
---

{% for post in site.posts %}
  {% include archive-single.html type="grid" %}
{% endfor %}