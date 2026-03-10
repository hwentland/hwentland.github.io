---
layout: default
title: Home
---

{% for post in site.posts %}
  <article>
    <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
    <p class="post-meta">{{ post.date | date: "%B %-d, %Y" }}</p>
    {% if post.excerpt %}
      {{ post.excerpt }}
    {% endif %}
  </article>
{% endfor %}

---

[RSS Feed](/feed.xml)
