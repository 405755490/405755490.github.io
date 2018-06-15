---
layout: page
title: 梦想成为架构师
titlebar: pboc
subtitle: <span class="mega-octicon octicon-keyboard"></span>&nbsp;&nbsp; 一步一步往上爬.
menu: pboc
css: ['blog-page.css']
permalink: /pboc
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='pboc' or post.category=='PBOC'  or post.keywords contains 'pboc' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
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
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });

    });
</script>
