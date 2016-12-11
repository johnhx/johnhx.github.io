---
layout: default
permalink: /archive/
title: "归档"
description: archive for blog posts
modified: 2015-07-28
tags: [archive]
---
<div class="page-content wc-container">
  <h1>归档</h1>
  {% for post in site.posts %}
    {% capture currentyear %}{{post.date | date: "%Y"}}{% endcapture %}
    {% if currentyear != year %}
        {% unless forloop.first %}</ul>{% endunless %}
            <h5>{{ currentyear }}</h5>
            <ul class="posts">
            {% capture year %}{{currentyear}}{% endcapture %}
        {% endif %}
    <li><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></li>
{% endfor %}
</div>