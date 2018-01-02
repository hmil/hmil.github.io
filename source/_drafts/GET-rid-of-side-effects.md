---
title: GET rid of side effects
id: 196
categories:
  - Uncategorized
tags:
---

While for some this may seem obvious, it's worth reminding it one more time: NEVER ever create a GET endpoint with side effects on your web server. This is an overlooked security principle and even if you know this principle, you may inadvertently create one.

- browser prefetching
- csrf not in url (if any csrf !)