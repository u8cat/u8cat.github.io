---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* B.E. in Computer Science & Technology, University of Science and Technology of China, 2024
* High School, Shenzhen Middle School, 2017

Work experience
======

* Summer & Fall 2023: Research Assistant
  * University of Michigan
  * Duties included: Conducting research on SQL verification
  * Supervisor: Prof. Xinyu Wang

Skills
======
* Computer Languages:
  * General Purpose Languages: Python, C++, Go, C, OCaml, Rust
  * Proof Assistants: Rocq, Lean
  * Hardware Description Languages: Verilog
  * Assembly Languages: RISC-V, ARMv7, LoongArch, AMD64
  * Other Languages: Antlr, LaTeX, LLVM, Yacc (Bison)
* Natural Languages: English (fluent), Mandarin (native)

Publications
======
  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

Talks
======
  <ul>{% for post in site.talks reversed %}
    {% include archive-single-talk-cv.html  %}
  {% endfor %}</ul>

Teaching
======
  <ul>{% for post in site.teaching reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

Service and leadership
======
* Student Volunteer in USTC CS [Debugger Salon](https://cs.ustc.edu.cn/2024/0423/c3058a639173/page.htm), Spring 2023 & Spring 2024

Misc
=====

- My preferred timezone is Coordinated Universal Time (UTC).
- My preferred temperature unit is Kelvin (K).
- My name in native alphabet is 张子辰 (U+5F20 U+5B50 U+8FB0).
