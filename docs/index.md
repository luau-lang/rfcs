# RFCs

{% assign sortedPages = site.pages | sort: 'title' %}

Luau uses RFCs to discuss and document the language evolution process. They are hosted [in a GitHub repository](https://github.com/luau-lang/rfcs) which accepts pull requests for corrections to existing RFCs as well as new RFCs.

If you'd like to submit a benign (grammatical, stylistic) correction, please open a pull request; all RFCs are in [docs folder in the repository](https://github.com/luau-lang/rfcs/tree/master/docs).

If you'd like to submit a new RFC, please [read our guidelines first](https://github.com/luau-lang/rfcs/blob/master/README.md); RFCs often require a significant amount of design effort and precision to be considered.

Note that if you'd like to submit a substantial change to an existing RFC that is already implemented according to its Status field, you would need to submit a new amendment RFC instead of editing an existing one.

<ul>
{% for page in sortedPages %}
{% assign ext = page.name | split:'.' | last %}
{% if ext == 'md' and page.name != 'index.md' %}
<li>
<a href="{{ page.url }}">{{ page.title }}</a>
</li>
{% endif %}
{% endfor %}
