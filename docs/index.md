# RFCs

{% assign sortedPages = site.pages | sort: 'title' %}

<ul>
{% for page in sortedPages %}
{% assign ext = page.name | split:'.' | last %}
{% if ext == 'md' and page.name != 'index.md' %}
<li>
<a href="{{ page.url }}">{{ page.title }}</a>
</li>
{% endif %}
{% endfor %}
