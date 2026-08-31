---
layout: page
title: Categories
permalink: /categories/
---

<div class="category-cloud" id="category-cloud">
{% assign sorted_cats = site.categories | sort %}
{% for category in sorted_cats %}
  <button class="category-chip" data-category="{{ category[0] | slugify }}" onclick="toggleCategory(this)">{{ category[0] }}<span class="count">{{ category[1] | size }}</span></button>
{% endfor %}
</div>

{% for category in sorted_cats %}
<div class="category-section" data-cat="{{ category[0] | slugify }}">
  <h2>{{ category[0] }}</h2>
  <ul>
  {% for post in category[1] %}
    <li>
      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      <span class="cat-date">{{ post.date | date: "%b %Y" }}</span>
    </li>
  {% endfor %}
  </ul>
</div>
{% endfor %}

<script>
var activeCategory = null;

function filterByCategory(cat) {
  var allBtns = document.querySelectorAll('.category-chip');
  var allSections = document.querySelectorAll('.category-section');

  if (activeCategory === cat) {
    activeCategory = null;
    window.history.replaceState(null, '', window.location.pathname);
    allBtns.forEach(function(b) { b.classList.remove('active'); });
    allSections.forEach(function(s) { s.style.display = ''; });
  } else {
    activeCategory = cat;
    window.history.replaceState(null, '', '#' + cat);
    allBtns.forEach(function(b) {
      b.classList.toggle('active', b.getAttribute('data-category') === cat);
    });
    allSections.forEach(function(s) {
      s.style.display = s.getAttribute('data-cat') === cat ? '' : 'none';
    });
  }
}

function toggleCategory(btn) {
  filterByCategory(btn.getAttribute('data-category'));
}

(function() {
  var hash = window.location.hash.replace('#', '');
  if (hash) filterByCategory(hash);
})();
</script>
