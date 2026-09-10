---
title: "Blogs"
layout: splash
permalink: /blogs/
---

<div class="notes-page">
<div class="notes-index">
  <header class="notes-index__header">
    <h1 class="notes-index__title">Blogs</h1>
    <p class="notes-index__lead">What is bro talm bout</p>
  </header>

  {% assign note_list = site.pages | where_exp: "p", "p.path == '__no_match__'" %}
  {% for coll in site.collections %}
    {% if coll.label == "blogs" %}
      {% assign note_list = coll.docs | sort: "date" | reverse %}
    {% endif %}
  {% endfor %}

  {% comment %} Beyond Temperature is a static page (not a collection doc), so render it
     as a card inserted into the date-sorted loop rather than pinned to the top. {% endcomment %}
  {% capture bt_card %}<article class="notes-card">
      <a class="notes-card__link" href="{{ '/blogs/beyond-temperature/' | relative_url }}" aria-label="Open blog: Logit transforms beyond temperature"></a>
      <div class="notes-card__inner">
        <h2 class="notes-card__title">Logit transforms beyond temperature</h2>
        <p class="notes-card__desc">Try replacing temperature (single scalar) with a piecewise monotonic function</p>
        <div class="notes-card__meta">
          <time class="notes-card__date" datetime="2026-05-15">15 May 2026</time>
          <span class="notes-card__hashtag">#llm</span>
        </div>
      </div>
    </article>{% endcapture %}
  {% assign bt_ts = "2026-05-15" | date: "%s" | plus: 0 %}
  {% assign bt_done = false %}

  <div class="notes-index__list">
    {% for note in note_list %}
      {% assign note_ts = note.date | date: "%s" | plus: 0 %}
      {% if bt_done == false %}{% if note_ts < bt_ts %}{{ bt_card }}{% assign bt_done = true %}{% endif %}{% endif %}
      <article class="notes-card">
        {% if note.external_url %}
          <a class="notes-card__link" href="{{ note.external_url }}" aria-label="Open blog: {{ note.title | strip | escape }}"></a>
        {% else %}
          <a class="notes-card__link" href="{{ note.url | relative_url }}" aria-label="Open blog: {{ note.title | strip | escape }}"></a>
        {% endif %}
        <div class="notes-card__inner">
          <h2 class="notes-card__title">{{ note.title }}</h2>
          {% assign blurb = note.description | default: note.excerpt %}
          {% if blurb %}
            <p class="notes-card__desc">{{ blurb | strip_html | truncate: 200 }}</p>
          {% endif %}
          <div class="notes-card__meta">
            {% if note.date %}
              <time class="notes-card__date" datetime="{{ note.date | date_to_xmlschema }}">{{ note.date | date: "%-d %B %Y" }}</time>
            {% endif %}
            {% if note.tags %}
              {% for t in note.tags %}<span class="notes-card__hashtag">#{{ t }}</span>{% endfor %}
            {% elsif note.tag %}
              <span class="notes-card__hashtag">#{{ note.tag }}</span>
            {% endif %}
          </div>
        </div>
      </article>
    {% endfor %}
    {% unless bt_done %}{{ bt_card }}{% endunless %}
  </div>
</div>

<p class="notes-page__footnote">Thanks for stopping by :)</p>

</div>
