---
layout: page
title: Blog Posts
author: Jae
---

<ul class="posts">
    {% for post in site.posts %}
    	<li> 
    		<span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}" style="text-decoration:none">{{ post.title }}</a>
		</li>
    {% endfor %}
</ul>

