---
layout: default
---

# Aleksandar Kircanski

<img src="images/smurf2.jpg" width="150" height="110" alt="hi" class="inline"/>

Hi, welcome to my homepage! My interests include security bug hunting, cryptography and software development.

Blog posts:

<ul class="posts">
{% for post in site.posts %}
  <li><span class="hero">{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}

<br></br>

Reading <a href="./reading-log">log</a> <3


