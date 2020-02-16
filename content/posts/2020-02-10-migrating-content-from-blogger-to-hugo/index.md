---
title: "Migrating Content from Blogger to Hugo"
date: 2020-02-10T13:56:12Z
draft: false
authors: ["martincjarvis"]
comments: true
thumbnail: "/coding-gallery"
series: ["hugo"]
---
# Automatically Migrating from Blogger to Hugo

After selecting [Hugo](https://gohugo.io/) and [Netlify](https://www.netlify.com/) to host the blog, I wanted to get some content in quickly populate the Archive.  

Fortunately, blogger offer an easy XML export of all your content which can be processed to convert the content over to Hugo.  Due to time constraints, I wanted to do this quickly with minimal manual effort so an automatic script was in order.

I ideally wanted to convert all the old posts to markdown, but quickly found that my previous method of embedding code was not trivial to convert, so I settled for HTML format posts.

