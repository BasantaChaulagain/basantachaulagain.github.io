---
title: "Blog"
permalink: /blog/
# layout: posts-flat
entries_layout: grid
author_profile: true
classes: wide
published: true
---

{% assign layout_type = page.entries_layout | default: 'list' %}
<div class="entries-{{ layout_type }}">
  {% for post in site.posts %}
    {% include archive-single.html
       post=post
       type=layout_type
       show_excerpt=true
       show_teaser=true
    %}
  {% endfor %}
</div>

<a href="#page-title" class="back-to-top">
  {{ site.data.ui-text[site.locale].back_to_top | default: 'Back to Top' }} &uarr;
</a>