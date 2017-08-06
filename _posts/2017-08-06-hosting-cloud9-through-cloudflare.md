---
layout: post
comments: true
title: "Hosting your own Cloud9 instance through CloudFlare?"
fulltitle: ""
excerpt: ""
categories : 
- cloud
- notes
- howto
---

## Disable Auto Minify under Speed settings otherwise CloudFlare will start to minify the documents you open for editing!

I initially thought there was a bug with Cloud9's handling of line endings because files with both Windows and Unix endings, some files just kept opening with all the content on 1 line.

When I narrowed the file types down to just JS, HTML and CSS files, I realised the mistake - I use the free version of CloudFlare to hide my origin server IP (it is hosted at home) and it's Speed optimisation tools will automatically minify source code for your website. But when you are editing those kind of files in a browser based editor, it will minify those too!

It can end up a right mess if you accidentally save it...