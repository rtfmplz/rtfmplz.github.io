---
layout: default
title: 
author: Jae
---

<!-- profile-img id를 적용 -->
<img id="profile-img" src="/images/private/jae.jpeg" alt="jae" />

## Blog Posts

<ul id="home-ul">
    {% for post in site.posts %}
    	<li> 
    		{{ post.date | date_to_string }} <a href="{{ post.url }}">{{ post.title }}</a>
		</li>
    {% endfor %}
</ul>

