---
layout: default
title: Axemblr Blog
tagline: Apache™ Hadoop® on-demand.
---
{% include JB/setup %}

<div class="hero-unit" style="text-align: center;">
    <a href="http://axemblr.com"><img src="assets/images/axemblr-logo-transparent.png" alt="Axemblr Software Solutions Logo" width="25%"/></a>
    <h3 style="padding: 20px; padding-left: 0px;">Apache™ Hadoop® on-demand.</h3>
</div>

<ul class="unstyled">
    {% assign posts_collate = site.posts %}
    {% include JB/posts_collate %}
</ul>
