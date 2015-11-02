---
layout: post
title: Some notes on django
category: [tech]
tags: [python django]
---
Recently I've been using Django in my projects. Here are some notes and pitfalls.

### How to use i18n with javascript
django' s translation also works for javascript. To make the .mo files available to js,
we need to add some url rules, and also add that path to html files that needs loading translation files.

The only thing needs attention is that, when making translation files for js, we need to give `makemessages` a
`-d djangojs` option, otherwise it won't work. The generated file is with name djangojs.po compared to the normal name django.po.

In the `js_info_dict` that is passed to `javascript_catalog`, we need to specify the domain as `djangojs`. It we specified `django` as
domain, then po files of python and templates will be loaded, which may not be what we want.
