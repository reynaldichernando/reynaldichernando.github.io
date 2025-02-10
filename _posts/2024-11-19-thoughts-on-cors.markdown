---
layout: post
title: "Thoughts on CORS"
date: 2024-11-19 15:16:15 +0700
categories: blog
author: Reynaldi Chernando
---

If you have worked in web development, you are probably familiar with CORS and have encountered this kind of error:

![CORS Error](https://imgur.com/eIP7KJC.png)
_CORS Error_

CORS is short for Cross-Origin Resource Sharing. It's basically a way to control which origins have access to a resource. It was created in 2006 and exists for important security reasons.

The most common argument for CORS is to prevent other websites from performing actions on your behalf on another website. Let's say you are logged into your bank account on Website A, with your credentials stored in your cookies. If you visit a malicious Website B that contains a script calling Website A's API to make transactions or change your PIN, this could lead to theft. CORS prevents this scenario.

![](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*-OfKGocQlWzm76vv7fGbsA.png)
_Cross site attack (source: [Felipe Young](https://medium.com/@youngfelipe1/cors-error-c5a0fc265fd7))_

Here's how CORS works: whenever you make a fetch request to an endpoint, the browser first sends a preflight request using the OPTIONS HTTP method. The endpoint then returns CORS headers specifying allowed origins and methods, which restrict API access. Upon receiving the response, the browser checks these headers, and if valid, proceeds to send the actual GET or POST request.

![Preflight Request](https://mdn.github.io/shared-assets/images/diagrams/http/cors/preflight-correct.svg)
_Preflight request (source: [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS))_

While this mechanism effectively protects against malicious actions, it also limits a website's ability to request resources from other domains or APIs.

This limitation becomes particularly frustrating when building a client-only web apps. For example, when I was building my [standalone YouTube player web app](https://github.com/reynaldichernando/backtrack), I needed two simple functions: search (using DuckDuckGo API) and video downloads (using YouTube API). Both endpoints implement CORS restrictions. So what can we do?

One solution is to create a backend server that proxies/relays requests from the client to the remote resource. This is exactly what I did, by creating [Corsfix, a CORS proxy](https://corsfix.com) to solve these errors. However, there are other popular open-source projects like [CORS Anywhere](https://github.com/Rob--W/cors-anywhere) that offer similar solutions for self-hosting.

![Diagram](https://imgur.com/j1dtNFR.png)
_CORS Proxy relaying request to remote resource_

Although, some APIs like YouTube's video API, are more restrictive with additional checks for origin and user-agent headers (which are forbidden to modify in request headers). Traditional CORS proxies can't bypass these restrictions. For these cases, I have special header override capabilities in my CORS proxy implementation.

With CORS proxy, you can fetch data from an API you don’t control. It is a simple and effective way to get around the CORS restrictions and access the data you need.
