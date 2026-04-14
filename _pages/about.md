---
layout: archive
permalink: /
title: "Projects"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Software engineer focused on systems programming and AI infrastructure. Recent graduate from the University of Michigan with a BS in Computer Science.

Skills: C/C++, Python, Verilog/SystemVerilog, FastAPI, PyTorch, SQL

Professional Experience: Cantor Fitzgerald (Technology Intern, Summer 2024) · Lynred-USA (Engineering Assistant, Summer 2023)

---

{% include base_path %}

{% for post in site.portfolio %}
  {% include archive-single.html %}
{% endfor %}



