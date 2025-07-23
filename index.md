---
layout: default
---

<div class="post-list">
  {% for post in site.posts %}
    <div class="post-item">
      <h2 class="post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <div class="post-meta">
        <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
        {% if post.categories.size > 0 %}
        <span class="post-categories">
          | Categories: 
          {% for category in post.categories %}
            <a href="{{ site.baseurl }}/categories/#{{ category | slugify }}">{{ category }}</a>
          {% endfor %}
        </span>
        {% endif %}
      </div>
      <div class="post-excerpt">
        {{ post.excerpt }}
      </div>
      <a href="{{ post.url | relative_url }}" class="read-more">阅读全文 &raquo;</a>
    </div>
  {% endfor %}
</div>