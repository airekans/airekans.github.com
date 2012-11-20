---
layout: page
title: Life About Programming
tagline: printf("Hello, Programs!\n");
---
{% include JB/setup %}

{% for post in site.posts %}

<div style="border-style:solid; border-color:#EEE;padding:5px;-webkit-border-radius:6px;-moz-border-radius:6px;border-radius:6px;">
<h1 class="index-post-title">{{ post.title }}</h1> <em>posted on {{ post.date | date_to_string }}</em>
<hr/>

{{ post.content }}

</div>
<br/>
<br/>

{% endfor %}

