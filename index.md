---
layout: page
title: Life About Programming
tagline: printf("Hello, Programs!\n");
---
{% include JB/setup %}

{% for post in site.posts %}

<div style="border-style:solid; border-color:#EEE; padding:5px;">
<h1>{{ post.title }}</h1> <em>posted on {{ post.date | date_to_string }}</em>
<hr/>

{{ post.content }}

</div>
<br/>
<br/>

{% endfor %}

