
{% include header.md %}

Other posts: 

{% for post in site.posts %}
* [{{post.title}}]({{ post.url | prepend: site.baseurl }})
{% endfor %}
