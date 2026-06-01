---
layout: page
title: contributions
permalink: /contributions/
description: Projects I contribute to through teams and associations.
nav: true
nav_order: 4
---

{% assign sorted_contributions = site.contributions | sort: "importance" %}

<div class="projects contributions">
  <div class="row row-cols-1 row-cols-md-2">
    {% for contribution in sorted_contributions %}
      {% include contributions.liquid %}
    {% endfor %}
  </div>
</div>
