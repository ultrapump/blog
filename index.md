---
layout: default
title: Ultrapump Engineering
---

<div class="blog-grid">
  {% for post in site.posts %}
    <article class="post-tile" {% if post.image %}style="background-image: url('{{ post.image | relative_url }}');"{% endif %}>
      <a href="{{ post.url | relative_url }}" class="tile-link">
        <time class="tile-date" datetime="{{ post.date | date: '%Y-%m-%d' }}">{{ post.date | date: '%B %d, %Y' }}</time>
        <div class="tile-content">
          <h2 class="tile-title">{{ post.title }}</h2>
          <p class="tile-description">{{ post.description | default: post.excerpt | strip_html | truncate: 120 }}</p>
        </div>
      </a>
    </article>
  {% endfor %}
</div>


