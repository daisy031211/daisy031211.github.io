---
layout: archive
title: "CTF / Wargame"
permalink: /ctf-wargame/
author_profile: true
---

{% assign category_posts = site.posts | where_exp: "post", "post.categories contains 'ctf-wargame'" %}

<div class="entries-grid">
  {% for post in category_posts %}
    {% include archive-single.html type="grid" %}
  {% endfor %}
</div>

{% if category_posts.size == 0 %}
아직 작성된 게시글이 없습니다.
{% endif %}
