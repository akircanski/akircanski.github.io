---
layout: default
---

# Aleksandar Kircanski

<img src="images/smurf2.jpg" width="100" height="70" alt="hi" class="inline"/>

Hi, welcome to my homepage! My interests are security bug hunting, software development and cryptography. 

Blog posts:

<ul class="posts">
{% for post in site.posts %}
  <li><span class="hero">{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}

