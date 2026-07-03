---
title: "Notes"
layout: splash
permalink: /notes/
---

<div class="notes-page">
<div class="notes-index">
  <header class="notes-index__header">
    <h1 class="notes-index__title">Notes</h1>
    <p class="notes-index__lead">Some things I think are cool! Currently mainly for my own reference, so heads up and apologies for poor writing, but hopefully it will get better.</p>
  </header>

  {% assign note_list = site.pages | where_exp: "p", "p.path == '__no_match__'" %}
  {% for coll in site.collections %}
    {% if coll.label == "notes" %}
      {% assign note_list = coll.docs | sort: "date" | reverse %}
    {% endif %}
  {% endfor %}

  {% comment %} Beyond Temperature is a static page (not a collection doc), so render it
     as a card inserted into the date-sorted loop rather than pinned to the top. {% endcomment %}
  {% capture bt_card %}<article class="notes-card">
      <a class="notes-card__link" href="{{ '/notes/beyond-temperature/' | relative_url }}" aria-label="Open note: Beyond Temperature"></a>
      <div class="notes-card__inner">
        <h2 class="notes-card__title">Beyond Temperature: Token-Adaptive Logit Transformations Learned with RLVR</h2>
        <p class="notes-card__desc">A final-project blog on learning per-token logit transformations with RLVR to escape the accuracy&ndash;diversity tradeoff that temperature sampling is stuck with.</p>
        <div class="notes-card__meta">
          <time class="notes-card__date" datetime="2026-05-15">May 15, 2026</time>
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
        <a class="notes-card__link" href="{{ note.url | relative_url }}" aria-label="Open note: {{ note.title | strip | escape }}"></a>
        <div class="notes-card__inner">
          <h2 class="notes-card__title">{{ note.title }}</h2>
          {% assign blurb = note.description | default: note.excerpt %}
          {% if blurb %}
            <p class="notes-card__desc">{{ blurb | strip_html | truncate: 200 }}</p>
          {% endif %}
          <div class="notes-card__meta">
            {% if note.date %}
              <time class="notes-card__date" datetime="{{ note.date | date_to_xmlschema }}">{{ note.date | date: "%b %d, %Y" }}</time>
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

<h3 class="notes-page__subhead" id="draft-topics">Draft topics</h3>
<ul class="notes-page__list">
  <li><em>TBD</em></li>
</ul>
<p class="notes-page__footnote">Thanks for stopping by :)</p>

</div>
