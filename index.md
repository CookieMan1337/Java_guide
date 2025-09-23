## Все статьи Spring

<ul>
  {% assign spring_pages = site.pages | where_exp: "p", "p.path contains 'spring/'" %}
  {% for p in spring_pages %}
    <li><a href="{{ p.url | relative_url }}">{{ p.title }}</a></li>
  {% endfor %}
</ul>
