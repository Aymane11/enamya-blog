---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
author: "Aymane BOUMAAZA"
draft: false
description: "{{ .Name }}"
tags:
 - "{{ .Name }}"
---