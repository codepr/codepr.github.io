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
  </div>
{% endfor %}
