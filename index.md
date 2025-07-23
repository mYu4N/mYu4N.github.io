
---
layout: default
---

<div class="main-content-wrap">
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
</div>

<!-- 我们也把首页列表的样式直接注入到这里，确保生效 -->
<style>
  .main-content-wrap {
      max-width: 800px;
      margin: 0 auto;
      padding: 2em;
  }
  .post-list .post-item {
    margin-bottom: 2.5em;
    padding-bottom: 2.5em;
    border-bottom: 1px solid #eee;
  }

  .post-list .post-item:last-child {
    border-bottom: none;
    margin-bottom: 0;
    padding-bottom: 0;
  }

  .post-list .post-title a {
    text-decoration: none;
    color: #333;
    transition: color 0.2s ease-in-out;
  }

  .post-list .post-title a:hover {
    color: #007bff;
  }

  .post-list .post-meta {
    color: #888;
    font-size: 0.9em;
    margin-top: 0.5em;
  }

  .post-list .post-meta a {
    color: #888;
    text-decoration: none;
  }

  .post-list .post-meta a:hover {
    text-decoration: underline;
  }

  .post-list .post-excerpt {
    margin-top: 1em;
    line-height: 1.6;
  }

  .post-list .read-more {
    display: inline-block;
    margin-top: 1em;
    text-decoration: none;
    font-weight: bold;
    color: #007bff;
  }
</style>
