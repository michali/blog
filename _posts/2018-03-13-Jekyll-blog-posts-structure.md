---
layout: post
title: "Jekyll blog posts in their own sub URL"
date: 2018-3-13 00:00
categories:
- Jekyll
excerpt: When rolling out a new blog, you can put all blog posts under their own sub URL.
---
As we know, by default, Jekyll creates a folder and URL structure for all blogs and pages on the site to exit alongside each other in a flat structure. So, the URL structure for this would be as follows:

```
.
├── blog post 1 permalink
├── blog post 2 permalink
├── about
├── projects
```

If your whole site is built in Jekyll, however, you may want to organise your URL structure a bit differently. Perhaps you'd like to have all of your posts under their own "sub-URL" and the pages on the root above, like below:

```
.
├── blog
     ├── post 1 permalink
     ├── post 2 permalink
├── about
├── projects
```

That would be something similar to

```
http://www.example.com/blog/2018/03/20/post-title-1
http://www.example.com/blog/2018/03/21/post-title-2
http://www.example.com/about
http://www.example.com/projects
```

_The solution below is for new blogs that are just coming out and haven't been out for long to have had the chance to have posts bookmarked etc. Existing blogs will have to have their old URLs redirected._

To achieve this you need to do three things:

## Folder with index file
To have an page with an index of your latest posts, under the new URL structure, create an index.html page in a new folder with the same name as the start the sub-URL. Jekyll will parse this and include it in the site

<div class="warning">If you are using jekyll-paginate in your blog post index page you must change the paginate_path in your _config.yml to include the sub-url prefix too. This also applies to jekyll-archives as they will also need to have their permalinks section in _config.yml prepended with the sub-url prefix.</div>

## Permalinks
Change the permalinks to your blog posts. That should exist in the Front Matter of each post or in your `_config.yml`. You add the sub-url to the permalink so that if your permalink template looked like this:

```
permalink: /:year/:month/:day/:title/
```

It becomes this:

```
permalink: /blog/:year/:month/:day/:title/
```

If you didn't have a permalink configuration the default of `/:categories/:year/:month/:day/:title:output_ext` would have been used so the template in the config file should look like `/blog/:categories/:year/:month/:day/:title:output_ext`

## Add the blog index to the site's navigation
We will assume your site has links to navigate around it. As the new index.html file is not a Jekyll page, it will not appear in the navigation section. To do this, I've created a YAML file called `nav.yml` in Jekyll's `_data` folder.

<div class="info">If a _data folder doesn't exist on your site create one in your project's root folder. This is the folder where you can store metadata for your site and access it at runtime</div>

The contents of my `nav.yml` file:

```
-
 name: Blog
 url: '/blog/'
```

Then, on your site's navigation links you iterate through the `nav.yml` entries (just one in this instance):

{% highlight html %}{% raw %}
{% for link in site.data.nav %}
  <!-- add a link template as you would do for your normal pages  -->
{% endfor %}
{% endraw %}{% endhighlight  %}

This can be places next to the iterator that picks up your normal pages.

Now your blog post URLs will be in a seperate sub-URL from your pages with a link to get to the post index.