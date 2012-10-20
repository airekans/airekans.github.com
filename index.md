---
layout: page
title: Life About Programming
tagline: Supporting tagline
---
{% include JB/setup %}

{% for post in site.posts %}

<div style="border-style:solid; border-color:#EEE; padding:5px;">
<h1>{{ post.title }}</h1> -- {{ post.date | date_to_string }}
<hr/>

{{ post.content }}

</div>
<br/>
<br/>

{% endfor %}

