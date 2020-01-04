---
author: Carlos Mendible
categories:
- dotNet
date: "2010-02-23T15:17:00Z"
description: Running Apache behind an IIS server. .Net Solution
# tags: ["Apache", "IIS", "UrlRewrite"]
title: Running Apache behind an IIS server. .Net Solution
url: /2010/02/23/running-apache-behind-an-iis-server-net-solution/
---
Yesterday I had the challenge to redirect &#8211; rewrite an url from my public IIS7 server to an internal Apache server. For instance a request to **http://www.mysite.com/app** handled by IIS should be redirected (using a reverse proxy) to ; **http://127.0.0.1:8080/app** where the internal Apache is listening.

Trying to configure it with the new [Application Request Routing (ARR)](http://www.iis.net/expand/ApplicationRequestRouting) module from Microsoft was a disappointing task. Once installed, my server stopped serving pages, and I could not find a way to make it work.

So I started googling for another solution, finding out that most of the internet articles tell you that is almost impossible to do the work the way ; I wanted, recommending to use Apache as your public server and then redirect &#8211; rewrite requests to an internal IIS server.

Finally I crossed my eyes over a page that talked about [Managed Fusion URL Rewriter](http://www.managedfusion.com/products/url-rewriter/) that was exactly what I was looking for: a simple, fast and easy to configure solution.

[Managed Fusion URL Rewriter](http://www.managedfusion.com/products/url-rewriter/) is a really small ASP.NET assembly based on Apache mod_rewrite which provides nice url rewrite and **reverse proxy** capabilities.

So to solve my issue I created an ASP.Net application to handle requests for **http://www.mysite.com/app** on my IIS server, following the examples provided with [Managed Fusion URL Rewriter](http://www.managedfusion.com/products/url-rewriter/) download, ; and changing the provided **ManagedFusion.Rewriter.txt** configuration file to:

``` powershell
RewriteEngine On
#RewriteLog c:log.txt ;
#RewriteLogLevel 9

RewriteBase /app
RewriteRule ^(.*)$ ; ; ; http://127.0.0.1:8080/app$1 [QSA, P]
```

The configuration simply tells to the [Managed Fusion URL Rewriter](http://www.managedfusion.com/products/url-rewriter/) that any call to **/app** should be redirected to the internal Apache server.

Hope it helps!!!