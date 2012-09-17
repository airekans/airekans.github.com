---
layout: page
title: Life About Programming
tagline: Supporting tagline
---
{% include JB/setup %}

{% for post in site.posts %}
--------

# {{ post.title }}

{{ post.content | strip_html | truncatewords: 55 }}

[Read more...]({{ post.url }})
{% endfor %}


