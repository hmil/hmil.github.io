---
title: The good software manifesto
id: 257
categories:
  - Uncategorized
tags:
---

*   Performant? lol no
*   Works

    *   ~100% unit test (quote if you don't test it you assume you don't need it)
    *   100% happy path e2e
    *   Most thinkable corner cases
    *   Use TDD where applicable
    *   Layers of tests for different purposes
    *   Rather than trying to be Dev/Ops, be Dev/QA and Ops/QA because you should not need a separate QA team. Maybe go on about automating jobs and how QA is the next one after Uber drivers

*   Maintainable

    *   Testing!!! -&gt; prevents the next person to break things without knowing
    *   Factorize like crazy
    *   Do not factorize (ie. not from the start)
    *   Use and abuse layers of abstraction (eg. top level component tests should read like poetry, put an example with/without PoM but do not explain what a PoM is because that can fit in another article)
