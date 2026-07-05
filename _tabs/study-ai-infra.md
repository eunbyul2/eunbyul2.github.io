---
layout: page
title: AI 인프라 구성 배포 스터디
icon: fas fa-server
order: 5
---

> 모각코 - "AI 시대에 개발자가 알아야 할 인프라 구성 배포" 스터디 기록

{% assign posts = site.categories["인프라"] | sort: 'date' | reverse %}
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
