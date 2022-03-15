---
layout: page
permalink: categories
title: HackTheBox Writeups
---
<div id="archives" class="post">
{% for category in site.categories %}
  <div class="archive-group">
    {% capture category_name %}{{ category | first }}{% endcapture %}
    <p>{% assign cat = site.data.categories[category_name] %}</p>
    
    <div id="#{{ category_name | slugize }}"></div>
    <h3 class="category-head" style="margin-bottom: 5px; margin-top: 10px">{{ category_name }}</h3>
    <p style="margin-bottom: 10px">{{ cat.description }}</p>
    
    <ul style="margin-bottom: 40px">
    {% for post in site.categories[category_name] %}
    
    <article class="archive-item">
      <li><h4 style="margin-left: 0px"><a href="{{ site.baseurl }}{{ post.url }}">{{post.title}}</a></h4></li>
    </article>
    
    {% endfor %}
    </ul>
  </div>
{% endfor %}
</div>
