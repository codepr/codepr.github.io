---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
title:
---
{% for post in site.posts %}
  <div id="post-short">
    <span class="date"> {{ post.date | date: "%Y %b %d" }}</span>&nbsp;&nbsp;<a href="{{site.url}}{{site.baseurl}}{{post.url}}"> {{ post.title }} </a>
    <!-- <a href="{{site.url}}{{site.baseurl}}{{post.url}}"> -->
    <!--   <h3>{{post.title}}</h3> -->
    <!-- </a> -->
    <!-- <i>posted on {{ post.date | date: "%-d %b %Y" }}</i> -->
    <!-- <p> -->
    <!--   {% if post.excerpt %} -->
    <!--     {{ post.excerpt }} -->
    <!--   {% else %} -->
    <!--     {{ post.content }} -->
    <!--   {% endif %} -->
    <!-- </p> -->
  </div>
{% endfor %}
