---
layout: default
title: Axmblr Blog
tagline: How to put The Yellow Elephant up in the Cloud
---
{% include JB/setup %}

<div class="hero-unit" style="text-align: center;">
    <a href="http://axemblr.com"><img src="assets/images/axemblr-logo-transparent.png" alt="Axemblr Software Solutions Logo" width="15%"/></a>
    <h3 style="padding: 20px; padding-left: 0px;">How to put the <a href="http://hadoop.apache.org/">Yello Elephant</a> Up! in the Cloud</h3>
</div>

<ul class="unstyled">
    {% assign posts_collate = site.posts %}
    {% include JB/posts_collate %}
</ul>
