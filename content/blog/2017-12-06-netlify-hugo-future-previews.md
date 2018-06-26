---
title: Building future dated previews with Hugo and Netlify
description: |
  You can use the netlify.toml file in your repository to override Netlify build settings, allowing you to generate future posts for certain branches.
date: 2017-12-06
tags:
- hugo
- netlify
url: /blog/netlify-hugo-future-previews
---

Hugo allows you to future date posts and specify whether you want to generate those future dated posts. It does so via a combination of [front matter](https://gohugo.io/content-management/front-matter/) and command line options for the [hugo command](https://gohugo.io/commands/hugo/).

## How front matter controls publication date

As an example, I want to publish this blog post on 6 December 2017, but I am writing it on 5 December 2017. Hugo has two different front matter variables which control the publication date of a post. 

The first one is the `date` variable. This variable defines the **date and time** at which the content was **created**. This is what will typically be displayed as the date and time of a blog post.

The second variable is `publishDate`. This is the **date and time** when the content will be **published**. If you run the Hugo generator before this date and time, then no content will be generated for that particular post.

Now, if either of these variables are not defined, then Hugo will use the value of the other one. As an example, here is the front matter for this particular blog post:

```yaml
title: Building future dated previews with Hugo and Netlify
date: 2017-12-06
tags:
- hugo
- netlify
url: /blog/netlify-hugo-future-previews
```

Since I only specified the `date` as 6 December 2017, the `publishDate` will also default to 6 December 2017.

## Generating future content

Since I am writing this blog post on 5 December, running the Hugo generator will not generate the content for this blog post. I can however instruct Hugo to generate future dated posts by passing the `-F` or `--buildFuture` option to the Hugo generator:

![Generate future dated posts](/assets/images/2017-12-06-netlify-hugo-future-previews/generate-future-posts.png)

As you can see, Hugo indicated that it generated 1 future post. And sure enough, if I browse the website locally, you can see that this blog post was generated even though it is dated for the following day:

![Future dated blog post generated](/assets/images/2017-12-06-netlify-hugo-future-previews/future-dated-generated.png)

## Instructing Netlify to generate future posts

In my [blog post from yesterday](/blog/why-i-like-netlify) I indicated [deploy previews](https://www.netlify.com/docs/continuous-deployment/#branches-deploys) as one of the reasons why I like Netlify. In short, Netlify will automatically build previews for branches or pull requests.

The only problem is that Netlify will not, by default, instruct the Hugo generator to generate future dated posts. This makes sense for your normal website, but for branches and pull request you may often future date blog posts since the branch or PR will only be merged at that future date. 

It turns out that Netlify has a way to override the Hugo generator settings in specific scenarios thanks to the concept of [Deploy Contexts](https://www.netlify.com/docs/continuous-deployment/#deploy-contexts). 

From the Netlify documentation:

> Deploy contexts are a way to tell Netlify how to build your site. They give you the flexibility to configure your site’s build depending on the context they are going to be deployed to.

So as per the documentation, there are three different contexts, namely **production** (for the main site deployment), **deploy-review** (for pull request deploy previews) and **branch-deploy** (for branch deploy previews). 

You can override certain site settings for these contexts in the `netlify.toml` file which you can include in the root folder of your Git repository. In this file you can override the `command` for generating the website. 

As you can see in the `netlify.toml` for my website below, I have specified the command for the **deploy-review** and **branch-deploy** contexts to pass the `--buildFuture` option to the Hugo generator.

```toml
[context.production.environment]
  HUGO_VERSION = "0.30.2"

[context.deploy-preview]
  command = "hugo --buildFuture"
  [context.deploy-preview.environment]
    HUGO_VERSION = "0.30.2"

[context.branch-deploy]
  command = "hugo --buildFuture"
  [context.branch-deploy.environment]
    HUGO_VERSION = "0.30.2"
```  

If you're curious, I use the **HUGO_VERSION** environment variable in the `netlify.toml` to specify the version of Hugo that Netlify should use to build my website. This way I can ensure that the version used by Netlify and the version I am running locally are the same and not get any weird surprises.

With that in place I can push the branch that contain this blog post to GitHub and Netlify will build a preview for it. As you can see in the Deploy Log for the preview, Netlify has executed the Hugo command with the `--buildFuture` option:

![Netlify Deploy Log](/assets/images/2017-12-06-netlify-hugo-future-previews/netlify-preview-deploy-log.png)

And yet again, if I look at that preview in my browser you will notice that this blog post, which is only dated for the following day at the time of writing it, is generated along with the other blog posts:

![Future dated blog post generated by Netlify](/assets/images/2017-12-06-netlify-hugo-future-previews/netlify-future-dated-generated.png)
