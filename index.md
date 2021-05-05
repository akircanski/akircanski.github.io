---
layout: default
---

# Aleksandar Kircanski

![image](images/smurf2.jpg)

Hi, welcome to my homepage! I am a researcher interested in security bug hunting,
cryptography and software development.

Blog post list example

<ul class="posts">
{% for post in site.posts %}
  <li><span class="hero">{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}

