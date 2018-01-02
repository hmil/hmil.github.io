---
title: A cli tool to generate badges
id: 17
categories:
  - JavaScript
date: 2014-11-02 11:28:17
tags:
---

You may have noticed how CI badges are taking over on github. By CI badge, I mean this kind of things: ![[build passing]](https://raw.githubusercontent.com/hmil/node-project-badge/master/images/build-success.png)
Build status, code coverage, package version, ... There are badge for about everything.

I wanted these badges for a school project where we had our own jenkins server running somewhere. The thing is, all of these badges come with the associated SaaS (like [travis](http://travis-ci.org "travis-ci") or [coveralls](https://coveralls.io/ "coveralls")). I wanted a CLI tool to generate these on my server from a bash script and I couldn't find that so I built it.

* * *

The goal was to have one tool able to generate any kind of badge with a unified style and little configuration. There would be shared style settings and different badge _kinds_ which would inherit these settings. Each _kind_ would then take as few configuration as possible to cover all major use cases._ _It also had to be easily extendable so that anyone can add new _kinds_ of badge.

The solution I came up with was to render on an [html5 2D rendering context](https://developer.mozilla.org/docs/Web/API/CanvasRenderingContext2D). It is then possible to render badges in the browser or in node using the exact same code.

&nbsp;

From there, I just had to hook up [node-canvas](https://github.com/Automattic/node-canvas) with the existing library, translate CLI arguments to badge settings and that was it. A flexible CLI badge generator. I even added a few built-in configuration files to reduce the number of arguments required to generate popular badges.

The only problem is that node-canvas uses a native library called [cairo](http://cairographics.org/). This means you need to install this dependency separately. Also npm needs to build c code when you install the package and it can be a real pain depending on your platform.

* * *

&nbsp;

Detailed documentation and install instructions are available on github:

*   The CLI tool (requires node and cairo): [https://github.com/hmil/node-project-badge](https://github.com/hmil/node-project-badge)
*   The underling framework: [https://github.com/hmil/project-badge](https://github.com/hmil/project-badge)
They are both MIT-licensed. If you want to add built-in configs or badge kinds, just submit a pull request. All types of contribution are welcome.