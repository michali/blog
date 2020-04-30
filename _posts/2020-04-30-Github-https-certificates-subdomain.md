---
layout: post
title:  "Github Pages  offers HTTPS for one custom domain. However you can point more than one domain to your pages (with caveats)."
date:   2020-04-30 00:00
categories:
- Github
- HTTPS
tags:
- HTTPS
- Github
---

Back in 2018 Github began to offer free dedicated certificates on Github Pages. This was achieved by teaming up with Let's Encrypt, a certificate authority that provides free SSL/TLS certificates. Before that, if you wanted to serve your site through HTTPS you had to use a service like Cloudflare, point your domain to its name servers and enable HTTPS through its interface. 

This would work for most scenarios. However, because the certificates on Cloudflare's free plan are shared, that raises a matter of perception. Do I want to be seen sharing the same security certificate with virtual [neighbours](https://www.troyhunt.com/should-you-care-about-the-quality-of-your-neighbours-on-a-san-certificate/) you don't want to be seen with? Sure, the certificates are managed by Cloudflare and if you're looking at the Subject Alternative Name entries in a cert, that means you have a good idea how shared certificates work. Regardless, for the off chance that some may pass moral judgement on a site they're visiting based on the quality of said site's cert neighbours, many of us would like our own certificate, dedicated to a domain.

Github support pages have detailed instructions on how to obtain a dedicated certificate but I'd like to bring up a particular case.

Suppose you own a domain called example.com and a site on example.github.io and that you want to point both www.example.com and example.com to the Github pages site using HTTPS. If you hosted your site yourself and could manage your own certificates, you can have both example.com and www.example.com under a wildcard certificate with the apex example.com domain added as a Subject Alternative Name. Github Pages, however, doesn't work that way. When you enable SSL on your site's settings for a custom domain, a certificate is generated only for that custom domain exactly as it's specified in the settings and it won't cover variations of that domain.

However, you can still have your connections to your apex domain redirect to a subdomain specified in Github pages. To set up your site on your Github Pages repo on a subdomain, go your Github page repo's settings and add the fully qualified domain under "Custom Domain".

Then go to your domain registrar's DNS management. You need a `CNAME` entry for the subdomain to point to your `[name].github.io` domain. Then for the apex domain, create four A records that point to the four IP addresses for Github pages:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```
Alternatively, you can create an `ALIAS` or `ANAME` to your `[name].github.io` domain.

Refer to your domain registrar's documentation on how to do this.

And this takes us to the meat of this post. Having configured the entries above, typing the apex domain's address in your browser's address bar will redirect the browser with a 301 Moved Permanently status code to the Github Pages custom subdomain. Github pages will automatically handle this.

Caveats:
As mentioned, Github will generate a certificate only for the domain explicitly specified in the repo's settings. You'll get a warning if you try to connect to `https://example.com`. So, what happens when go to `example.com` is that you connect to Github Pages unsecurely and then Github Pages redirects the browser to the `www` domain. The initial connection before the redirect uses HTTPand thus it's vulnerable to man-in-the-middle attacks.

[HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) is not currently supported although Github is aware that there is a growing interest in it.

I guess you can’t complain, really. It’s all free, after all, save the price of the domain.