---
title: "Security Wiki"
layout: single
permalink: /wiki/
author_profile: false
---

<div class="wiki-directory">
  {% assign wiki_entries = site.wiki | group_by: "category" %}
  
  {% for category in wiki_entries %}
  
    <!-- Assign an image based on the category name -->
    {% assign cat_image = "/assets/images/default-folder.jpg" %}
    {% case category.name %}
      {% when 'Active Directory' %}
        {% assign cat_image = "/assets/images/AD.png" %}
      {% when 'Malware Analysis' %}
        {% assign cat_image = "/assets/images/thumb-malware.jpg" %}
      {% when 'DFIR' %}
        {% assign cat_image = "/assets/images/thumb-dfir.jpg" %}
    {% endcase %}
  
  <div class="folder-card">
    
    <!-- New Thumbnail Image Slot -->
    <img src="{{ cat_image }}" alt="{{ category.name }}" class="folder-thumb">
    
    <div class="folder-header">
      <i class="fas fa-folder-open folder-icon"></i>
      <h2 class="folder-title">{{ category.name }}</h2>
    </div>
    
    <ul class="file-list">
      {% for note in category.items %}
        <li>
          <a href="{{ note.url }}">
            <i class="fas fa-file-code file-icon"></i> {{ note.title }}
          </a>
        </li>
      {% endfor %}
    </ul>
    
  </div>
  {% endfor %}
</div>