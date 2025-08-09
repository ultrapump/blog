---
layout: default
title: Ultrapump Engineering
---

<div class="blog-grid">
  {% for post in site.posts %}
    <article class="post-tile">
      <a href="{{ post.url | relative_url }}" class="tile-link">
        <div class="tile-image">
          {% if post.image %}
            <img src="{{ post.image | relative_url }}" alt="{{ post.title }}" loading="lazy">
          {% else %}
            <div class="tile-placeholder">
              <span>{{ post.title | truncate: 20 }}</span>
            </div>
          {% endif %}
        </div>
        <div class="tile-content">
          <h2 class="tile-title">{{ post.title }}</h2>
          <p class="tile-description">{{ post.description | default: post.excerpt | strip_html | truncate: 120 }}</p>
          <time class="tile-date" datetime="{{ post.date | date: '%Y-%m-%d' }}">{{ post.date | date: '%B %d, %Y' }}</time>
        </div>
      </a>
    </article>
  {% endfor %}
</div>


