---
layout: default
title: "软件编程全部文章"
---
<ul class="list-unstyled">
{% for post in site.categories.program %}
  {% capture y %}{{post.date | date:"%Y"}}{% endcapture %}
  {% if year != y %}
    {% assign year = y %}
    <h2>{{ post.date | date: '%Y' }}</h2> 
  {% endif %}
  <li>
	<h4><span>{{ post.date | date:"%Y-%m-%d" }}</span>&raquo;
	<a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></h4>
  </li> 
{% endfor %}
</ul>
