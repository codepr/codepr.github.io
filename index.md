---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
title: Why hello there
---

Andrea here [GitHub](https://github.com/codepr) / [LinkedIn](https://www.linkedin.com/in/andrea-giacomo-baldan-000776aa) / [E-Mail](mailto:a.g.baldan@gmail.com)

## Archive

<div id="blog-group">
  {% for post in site.posts %}
    <div id="post-short">
      <span class="date"> {{ post.date | date: "%Y %b %d" }}</span>&nbsp;&nbsp;<a href="{{site.url}}{{site.baseurl}}{{post.url}}"> {{ post.title }} </a>
    </div>
  {% endfor %}
</div>

## Projects

Non-exhaustive list of open source pet projects I worked on for the last few
years, in no particular order and at various stages of development, mostly
didactic stuff and topics of curiosity.

- [Machines](https://github.com/codepr/machines){:target="_blank"}: playground for the development of simple compilers, VMs and various experiments.
- [Sol](https://github.com/codepr/sol.git){:target="_blank"}: lightweight MQTT broker from scratch
- [EV](https://github.com/codepr/ev.git){:target="_blank"}: lightweight event-loop library based on multiplexing IO
- [llb](https://github.com/codepr/llb.git){:target="_blank"}: dead simple event-driven load balancer
- [rlb](https://github.com/codepr/rlb.git){:target="_blank"}: rough-and-ready lightweight load-balancer in Rust
- [Roach](https://github.com/codepr/roach){:target="_blank"}: basic timeseries library and server with WAL and log-based persistence
- [SLS](https://github.com/codepr/sls){:target="_blank"}: bitcask basic implementation in Elixir
- [Rublo](https://github.com/codepr/rublo){:target="_blank"}: Bloom filter with barebone features in Rust
- [Tasq](https://github.com/codepr/tasq.git){:target="_blank"}: python task queue
- [Battletank](https://github.com/codepr/battletank){:target="_blank"}: Dumb raylib based implementation of arcade battletank
- [Aiotunnel](https://github.com/codepr/aiotunnel.git){:target="_blank"}: HTTP(S) tunneling for local and remote port-forwarding
- [Webcrawler](https://github.com/codepr/webcrawler.git){:target="_blank"}: Simple webcrawler, list on output all subdomains of a website
