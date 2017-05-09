---
layout: default
title: 
author: Jae
---

<!-- profile-img id를 적용 -->
<img id="profile-img" src="/images/private/jae.jpeg" alt="jae" />

## Blog Posts

<ul id="home-ul" class="posts">
    {% for post in site.posts %}
    	<li> 
    		<span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}" style="text-decoration:none">{{ post.title }}</a>
		</li>
    {% endfor %}
</ul>

