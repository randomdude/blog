
{% include header.md %}

## Welcome to GitHub Pages

Available posts:

{% for post in site.posts %}
* [{{post.title}}]({{ post.url | prepend: site.baseurl }})
{% endfor %}

