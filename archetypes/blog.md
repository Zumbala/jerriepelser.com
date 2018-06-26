---
title: "{{ replace .TranslationBaseName "-" " " | title }}"
date: "{{ now.Format "2006-01-02" }}"
tags:
-
url: /blog/{{ .TranslationBaseName }}
---
