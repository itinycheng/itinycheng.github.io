---
layout: default
title: "归档：Archives"
---
<div>
Tag:
	{% for tag in site.tags %} 
		<a href="tags.html#{{ tag[0] }}">{{ tag[0] }}</a>
		<ul>
		{% for post in tag[1] %}
			<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a>[{{ post.category }}]</li>
		{% endfor %}
		</ul>
	{% endfor %}

	
Category:
<ul>
	{% for category in site.categories %}
		<a href="/blog/{{ category[0] }}.html">{{ category[0] }}</a>
			{% for post in category[1] %}
			<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a>[{{ post.category }}]</li>
		{% endfor %}
	{% endfor %}
</ul>
</div>

