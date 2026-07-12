---
layout: page
title: Kubernetes 스터디
icon: fas fa-cubes
order: 5
---

> Kubernetes 스터디 기록

{% assign posts = site.posts | where_exp: "post", "post.path contains '_posts/kubernetes/'" %}
{% assign posts = posts | sort: 'date' | reverse %}
<div class="list-group">
{% for post in posts %}
  <a href="{{ post.url | relative_url }}" class="list-group-item list-group-item-action d-flex justify-content-between align-items-center">
    <span>{{ post.title }}</span>
    <small class="text-muted ms-3">{{ post.date | date: "%Y-%m-%d" }}</small>
  </a>
{% endfor %}
</div>
{% if posts.size == 0 %}
<p class="text-muted">아직 작성된 글이 없습니다.</p>
{% endif %}
