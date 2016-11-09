---
layout: archive
permalink: /tim/
title: "Tim's Articles"
date: 2016-10-24T21:40:00-00:00
modified: 2016-10-24T21:40:00-00:00
excerpt: "Making code simple and fast."
author_profile: true
author: "Tim Seaward"
header:
  overlay_image: timheader.png
  overlay_filter: rgba(50, 50, 50, 0.5)
---

{% for post in site.categories.tim %}
  {% include archive-single.html type="grid" %}
{% endfor %}