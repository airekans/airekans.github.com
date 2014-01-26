---
layout: post
title: Life About Programming
tagline: printf("Hello, Programs!\n");
index: true
---
{% include JB/setup %}

{% for post in site.posts limit:5 %}

<div class="index-post">
  <div class="index-post-header">
    <div><h1 class="index-post-title">{{ post.title }}</h1></div>
    <div style="color:grey;"><em>posted on {{ post.date | date_to_string }}</em></div>
  </div>
  <div>
  hello
  </div>
  <div class="index-post-link">
    <a href="{{ post.url }}">阅读全文...</a>
  </div>
</div>

{% endfor %}

<div style="text-align:right;">
  <a href="{{ site.JB.archive_path }}">More...</a>
</div>
