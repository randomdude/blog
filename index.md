
{% include header.md %}

## Welcome to my blog

Available posts:

{% for post in site.posts %}
* [{{post.title}}]({{ post.url | prepend: site.baseurl }})
{% endfor %}

