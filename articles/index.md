---
layout: home
---

<div class="index-content articles">
    <div class="section">
        <ul class="artical-cate">
            <li class="text-left"><a href="/"><span>Blog</span></a></li>
            <li class="text-center on"><a href="/articles"><span>Articles</span></a></li>        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.articles %}
            <li>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
