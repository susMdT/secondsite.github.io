---
layout: page
permalink: testing
title: HackTheBox Writeups TESTING
---
<div id="archives" class="post">
{% for category in site.categories %}
  <div class="archive-group">
    {% capture category_name %}{{ category | first }}{% endcapture %}  <!--- Get the first value of the category array, and set category_name value to it--->
    <p>{% assign cat = site.data.categories[category_name] %}</p>
    
    <div id="#{{ category_name | slugize }}"></div>
    <article class="post-preview">
    <h3 class="category-head" style="margin-bottom: 5px; margin-top: 10px">{{ category_name }}</h3> <!---  HackTheBox title --->
    <p style="margin-bottom: 10px">{{ cat.description }}</p> <!---  Nonexistant Description --->
    <ul style="margin-bottom: 40px">
    {% for post in site.categories[category_name] %} <!---  Individual Posts--->
      <article class="archive-item">
        <li><h4 style="margin-left: 0px"><a href="{{ site.baseurl }}{{ post.url }}">{{post.title}}</a></h4></li>
        <li><h4 style="margin-left: 0px"><a href="google.com">{{post.thumbnail-img}}</a></h4></li> <!---  Debug Output --->
        <div class="post-image post-image-normal">
        <a href="{{ post.url | absolute_url }}" aria-label="Thumbnail">
          <img src="https://dtsec.us{{post.thumbnail-img}}" alt="Post thumbnail">
        </a>
       </div>
      </article>
     </article>
    {% endfor %}
    </ul>
  </div>
{% endfor %}
</div>
