---
title: "{{ os.Getenv "HUGO_TITLE" | default (replace .File.ContentBaseName "-" " " | title) }}"
date: {{ .Date | time.Format "2006-01-02" }}
draft: true
description: ""
tags: []
---

<!-- Imagen de portada opcional: añade featured.png (o .jpg) junto a este archivo. -->

