---
layout: page
permalink: categories
title: HackTheBox Writeups
---
{{ content }}

<div id="archives" class="post">
{% for category in site.categories %}
  <div class="archive-group">
    {% capture category_name %}{{ category | first }}{% endcapture %}  
    <p>{% assign cat = site.data.categories[category_name] %}</p>
    
    <div id="#{{ category_name | slugize }}"></div>
    <h3 class="category-head" style="margin-bottom: 5px; margin-top: 10px">{{ category_name }}</h3> <!---  HackTheBox title --->
    {%- capture thumbnail -%}           <!---  Liquid, line 14 to 23 --->
          {% if post.thumbnail-img %}
        {{ post.thumbnail-img }}
      {% elsif post.cover-img %}
        {% if post.cover-img.first %}
          {{ post.cover-img[0].first.first }}
        {% else %}
          {{ post.cover-img }}
        {% endif %}
      {% else %}
      {% endif %}
    {% endcapture %}        <!---  Liquid, line 14 to 23 --->
    <p style="margin-bottom: 10px">{{ cat.description }}</p> <!---  Nonexistant Description --->
    
    <ul style="margin-bottom: 40px">
    {% for post in site.categories[category_name] %} <!---  Individual Posts--->
    
    <article class="archive-item">
      <li><h4 style="margin-left: 0px"><a href="{{ site.baseurl }}{{ post.url }}">{{post.title}}</a></h4></li>
    </article>
    
    {% endfor %}
    </ul>
  </div>
{% endfor %}
</div>
