---
layout: archive
permalink: /
title: "Background"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Software Engineer

BS Computer Science, University of Michigan

Professional Experience: Cantor Fitzgerald (Technology Intern, Summer 2024) · Lynred-USA (Engineering Assistant, Summer 2023)

---

## Projects

{% include base_path %}

{% assign sorted_portfolio = site.portfolio | sort: 'date' | reverse %}
{% for post in sorted_portfolio %}
  {% include archive-single.html %}
{% endfor %}



