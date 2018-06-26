---
title: Why I like Netlify
description: |
  I have switched my blog over to Hugo hosted on Netlify. Here are some great Netlify features I think you will like.
date: 2017-12-05
tags:
- hugo
- netlify
url: /blog/why-i-like-netlify
---

A few months back I switched my blog over from [Jekyll](https://jekyllrb.com/) hosted [GitHub Pages](https://pages.github.com/) to [Hugo](https://gohugo.io/) hosted on [Netlify](https://www.netlify.com/). 

The two main reasons were **speed** and **HTTPS**. 

More specifically, after five years' worth of blog posts, Jekyll stared to become a bit slow to regenerate the website. Initial tests with Hugo however proved that it was **very** quick. The other reason was that I was thinking of migrating over to HTTPS. This cannot be done out of the box when using a custom domain name on GitHub Pages, and instead you have to resort to [using something like Cloudflare](https://sheharyar.me/blog/free-ssl-for-github-pages-with-custom-domains/).

I stumbled across Netlify by chance one day and I decided to give them a try, and so far I am extremely happy with them. In this blog post I'll list eight features which I have used and really like.

## 1. Free

You cannot beat **Free**, especially when combined with all the other reasons I'll mention. They have [paid plans](https://www.netlify.com/pricing/), but those will typically only come into play once you start working in a team setup where you need multiple people to be able to access your account.

## 2. Fast

Combine a static website with their global CDN and what you get is a **super speedy** website. No matter from where in the world people visit your website, they will have a great experience.

## 3. Continuous deployment

It is very easy to configure [continuous deployment](https://www.netlify.com/docs/continuous-deployment/) from GitHub. And it works with plain static websites, static website generators (like Hugo) or even more advanced scenarios like using front-end build tools like NPM, Gulp or Grunt. Once you have configured the continuous deployment, you just need to commit to your GitHub repository and Netlify will take care of the rest.

## 4. Versioning and Rollbacks

Along with Continuous Deployment, you also get versioning and rollback. You can see a full history of all your deployments:

![View a list of all deploys](/assets/images/2017-12-05-why-i-like-netlify/all-deploys.png)

And even switch back to one of your old deployments if your current deployment proves problematic (this has happened to me before!). Simply hit the **Publish deploy** button on the build you want to revert to:

![Publish an old deployment](/assets/images/2017-12-05-why-i-like-netlify/publish-deploy.png)

## 5. Previews

Netlify will automatically build previews of branches and pull requests. Here you can see a preview build for the branch containing this blog post:

![Build deploy for a branch](/assets/images/2017-12-05-why-i-like-netlify/branch-deploy.png)

To view a preview of the deployment, simply click on the deployment in the list and then click the **Preview deploy** link:

![Preview branch deploy](/assets/images/2017-12-05-why-i-like-netlify/preview-deploy.png)

## 6. Asset optimization

Netlify can do asset optimization, such as bundling and minifying CSS and JavaScript and also compress images. For example, while developing you can include a whole bunch of CSS and JavaScript files. After building your website, Netlify will then bundle and minify all consecutive stylesheets and script references

![Asset optimization settings](/assets/images/2017-12-05-why-i-like-netlify/asset-optimization-settings.png)

Here are the network requests for CSS and JavaScript files when running my blog locally:

![Network requests when running locally](/assets/images/2017-12-05-why-i-like-netlify/assets-locally.png)

And here are the network requests for CSS and JavaScript files when running on Netlify. Notice that Netlify combined the CSS and JS files automatically:

![Network requests when deployed on Netlify](/assets/images/2017-12-05-why-i-like-netlify/assets-netlify.png)

## 7. Single-click SSL

Want to [configure your site to use SSL](https://www.netlify.com/docs/ssl/)? Click one button and Netlify will automatically provision a certificate from [Let's Encrypt](https://letsencrypt.org/). You can even force SLL for all requests, which will redirect from **HTTP** to **HTTPS** and add [Strict Transport Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) headers to all requests.

## 8. Redirect and Rewrite Rules

Netlify allows easy configuration of [redirect and rewrite rules](https://www.netlify.com/docs/redirects/) by adding a `_redirects` file to the root of your website where you can configure all the rules and optionally the HTTP Status codes, for example:

```text
/home              /                       302
/blog/my-post.php  /blog/my-post
/news              /blog
/google            https://www.google.com
```

## Some Others

Listed above are eight reasons why I like Netlify. While I have used all eight of those features, there are many more reasons why you may fall in love with Netlify. Here are a few more interesting features which I have not found use for yet, but which looks interesting:

* [Prerendering for Single Page Apps](https://www.netlify.com/docs/prerendering/) assists with SEO for SPA applications which are not behind a login screen.
* [Visitor Access Control](https://www.netlify.com/docs/visitor-access-control/) allows you to secure pages of your website using [JSON Web Tokens](https://jwt.io/). _This one looks very interesting to me. I think I need to explore this in a future blog post._
* [Split testing](https://www.netlify.com/docs/split-testing/) allows you to drive traffic to your website to different deploys. This is useful for tesing different designs, layouts, content, etc.

## Conclusion

I really like Netlify. There is not a single thing about it so far which has frustrated or irritated me. Instead, it just keeps on surprising me - in a pleasant way. If you are currently hosting a static website or Single Page Application somewhere else, then I suggest you give Netlify a try.