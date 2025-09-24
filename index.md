<ul>
  {% for doc in site.javacore %}
    <li><a href="{{ doc.url | relative_url }}">{{ doc.title }}</a></li>
  {% endfor %}
</ul>

<ul>
  {% for doc in site.spring %}
    <li><a href="{{ doc.url | relative_url }}">{{ doc.title }}</a></li>
  {% endfor %}
</ul>
