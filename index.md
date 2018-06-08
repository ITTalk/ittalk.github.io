---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
---

<div>
  {% for post in site.posts %}
    
      <div>
        <a href="{{ post.url | prepend: site.baseurl }}" class="post-title-link">
            <h1>{{ post.title }}</h1>
        </a>
        <p class="post-title-author">{{post.post_author}}, <time datetime="{{ post.date }}">{{ post.date | date: "%d %B %Y" }}</time></p>
        <p>
          {{ post.content | truncatewords: 50 | strip_html }}
        </p>
        <div class="post-separator"></div>
      </div>
  {% endfor %}
</div>