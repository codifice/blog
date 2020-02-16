---
title: "Rebooting the Blog"
date: 2020-02-10T09:26:39Z
draft: false
authors: ["martincjarvis"]
comments: true
thumbnail: "/coding-gallery"
series: ["hugo"]
---

# Rebooting the Blog

It's been a while since I've looked at the blog or felt I had anything to blog about (or could blog about).  But as my current contract approaches it's conclusion, I've got time to finally start contributing again - for a while at least!

## Out with the Old

My first order of business is a review of my blog [MartinOnDotNet](https://martinondotnet.blogspot.com/) which I host on Blogger and was mostly documenting feature/helpers etc around the Ektron and EPiServer CMS products which I found generally useful across multiple Client implementations. This brought back to mind some of the day-to-day issues that I faced:
* Awkward editing with images/code snippets and attachments
* No real change control
* Complex templating

I used a desktop tool to mitigate a lot of these issues, but ideally I want to be able to blog wherever I code with minimal fuss.  

## In with the New

Over the last 8(!) years, there's been a rise of static CMS systems such as [Jekyll](https://jekyllrb.com/) and [Hugo](https://gohugo.io/) which integrate into GitHub and will publish the site to GitHub Pages automatically on checkin.  

There are even services such as [Netlify](https://www.netlify.com/) that augment this functionality to provide previews based on feature branches, so you can safely preview template modifications without impacting the main site.  This means blogging will follow very similar flows to coding which is excellent.

{{< imgproc netlify-github Fill "650x300 bottom" >}}
Netlify and Github PR Integration
{{< / imgproc>}}

Using Markdown as the content format of choice is a no brainer, you just need a text editor and most code IDE's support previewing.

In my case looking at the options, both platforms would do more than what I want and I picked Hugo for a couple of (very minor) points:
* Hugo has a faster publish pipeline (but not by much)
* Hugo is based on Go (not ruby) and I've got a little bit more experience with Go