---
title: "Custom Google Search string to not show AI overview"
date: 2025-10-20T23:00:00-00:00
draft: false
---

I hate this AI overview thing. If I wanted to use a LLM, I would go use a LLM.

To get rid of it, you can create a custom search engine in your preferred browser with the following search string:

```txt
{google:baseURL}/search?udm=14&q=%s
```

`udm=14` takes you to the "web" results area immediately (not the primary one with AI garbage on it).

To add a new search engine (search string) in Chrome, navigate to `chrome://settings/searchEngines`, scroll down to "Site Search Engines", click Add, then click the three dots next to your new "search engine" and set it as the default.

Your searches should now take you right to web results, not the AI overview. Who knows how long this will last.
