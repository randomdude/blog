
Handy links:

* My [Twitter](https://twitter.com/alizthehax0r) 
* My [Mastodon](https://infosec.exchange/@alizthehax0r)

Other posts:

{% for post in site.posts %}
* [{{post.title}}]({{ post.url | prepend: site.baseurl }})
{% endfor %}

