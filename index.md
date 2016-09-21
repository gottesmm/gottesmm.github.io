---
layout: default
title: "mgottesman's swift working documents"
date: 2016-00-20 17:19:42 -0700
---

# Proposals

{% for p in site.pages %}
{{p.path}}
  {% if p.path.include? "proposal" %}
  - [{{p.title}}]({{p.url}})
  {% endif %}
{% endfor %}

