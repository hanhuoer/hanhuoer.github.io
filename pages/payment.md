---
layout: page
title: 支付系列文章
titlebar: payment
subtitle: <span class="class="mega-octicon octicon-clippy""></span>&nbsp;&nbsp; 支付、第三方支付系列文章
menu: payment
css: ['blog-page.css']
permalink: /payment
keywords: payment, 支付
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
            {% if post.category=='payment' or post.category=='支付' %}
            <li class="posts-list-item">
                <div class="posts-content">
                    <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                    <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title
                        }}</a>
                    <span class='circle'></span>
                </div>
            </li>
            {% endif %}
            {% endfor %}
        </ul>

        <!-- Pagination -->
        {% include pagination.html %}

        <!-- Comments -->
        <div class="comment">
            {% include comments.html %}
        </div>
    </div>

</div>
<script>
    $(document).ready(function () {

        // Enable bootstrap tooltip
        $("body").tooltip({selector: '[data-toggle=tooltip]'});

    });
</script>
