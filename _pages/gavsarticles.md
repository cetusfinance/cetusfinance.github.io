---
layout: archive
permalink: /gav/
title: "Gav's Articles"
date: 2016-10-24T21:40:00-00:00
modified: 2016-10-24T21:40:00-00:00
excerpt: "Math and complex problems explained in detail and a level of simplicity."
author_profile: true
header:
  overlay_image: gavheader.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
---

{% for post in site.categories.gav %}
  {% include archive-single.html %}
{% endfor %}