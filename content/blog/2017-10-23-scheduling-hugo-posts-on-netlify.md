---
title: Scheduling Hugo posts on Netlify
description: |
  A look at a couple of approaches you can use to schedule future posts using Netlify and Hugo.
date: 2017-10-23T00:00:00Z
publishDate: 2017-10-23T18:00:00+07:00
tags:
- hugo
- netlify
url: /blog/scheduling-hugo-posts-on-netlify
---

I recently switched by blog over from Jekyll running on GitHub Pages, to [Hugo](https://gohugo.io) running on [Netlify](https://gohugo.io).

One of the things I investigated was how to schedule blog posts on Netlify. This is, as with most other Static Website Generators, not something which is supported out of the box due to the fact that these websites are not dynamic.

The content is created when the static generator runs, so you need some way to re-generate the website at a future date and time to include those future-dated posts.

Before we dive into possibilities for future dating posts, let's take a quick look at how Netlify handles the building and publishing of a website using Continuous Integration.

## Continous Integration of a Hugo website from GitHub

When creating a new website on Netlify you can configure Continuous Deployment by connecting it to a repository hosted on GitHub (or GitLab or BitBucket).

![Continuous Deployment options when creating a new site on Netlify](/assets/images/2017-10-23-scheduling-hugo-posts-on-netlify/netlify-create-a-new-site.png)

Netlify will create a webhook which will be triggered every time you push changes to your GitHub repository. So given you have configured everything else correctly, all you need to do to publish new posts is to push changes to your GitHub repository.

When you do that, the webhook will be triggered, Netlify will pull the latest changes from GitHub, build your new Hugo website and publish it. 

Pretty straight forward.

Now let's look at the two approaches for working with future dated posts.

## Merging a Pull Request at a future date

The first approach is one which is commonly used with other platforms as well, and that is to merge a Pull Request at a future date and time.

When adding a new post to your Hugo website, you do not commit the changes to the `master` branch (or whatever branch Netlify is publishing from). Instead, create a new branch - either in the main repository or a [forked repository](https://help.github.com/articles/fork-a-repo/) - and commit your changes to that branch.

Once you are happy with the changes, you can then [create a pull request](https://help.github.com/articles/creating-a-pull-request/) for those changes to merged into the branch which Netlify is building from.

Netlify will build a preview site for the pull request, and you can click on the **Details** link which will take you to a preview version of the website where you can view the new post.

![Creating a pull request in GitHub with Netlify deploying a preview](/assets/images/2017-10-23-scheduling-hugo-posts-on-netlify/pull-request.png)

Now all you need to do is schedule for that pull request to be merged at some future date and time. You can use the [Merge a pull request](https://developer.github.com/v3/pulls/#merge-a-pull-request-merge-button) endpoint of the GitHub API to achieve this.

## Using the PublishDate Front Matter variable

The second approach is to specify the `publishDate` front matter variable for a post. When you do this, Hugo will not include the post in the generated content unless the date specified in the `publishDate` variable has already passed.

For example, lets say I want to publish this post at 6PM Bangkok time, on 23 October 2017. The time zone for Bangkok is UTC+7, so I can specify the `publishDate` using the [ISO 8601 date format](https://en.wikipedia.org/wiki/ISO_8601) for this date and time:

```
title: Scheduling Hugo posts on Netlify
date: 2017-10-23T00:00:00Z
publishDate: 2017-10-23T18:00:00+07:00
tags:
- hugo
- netlify
url: /blog/scheduling-hugo-posts-on-netlify
```

I can now commit this blog post and push the changes to GitHub. Netlify will use Hugo to build the website, but because that time has not yet passed, the post will not be included.

I will need to trigger the build process in Netlify again to do that.

In order to do that, you will first need to determine the registered hooks for the repository by using the [List Hooks](https://developer.github.com/v3/repos/hooks/#list-hooks) endpoint of the GitHub API.

This endpoint will return a JSON result similar to the following:

```json
[
    {
        "type": "Repository",
        "id": 11111111,
        "name": "web",
        "active": true,
        "events": [
            "delete",
            "pull_request",
            "push"
        ],
        "config": {
            "content_type": "json",
            "url": "https://api.netlify.com/hooks/github",
            "insecure_ssl": "0"
        },
        "updated_at": "2017-10-02T13:55:01Z",
        "created_at": "2017-10-02T13:55:01Z",
        "url": "https://api.github.com/repos/jerriep/jerriepelser.com/hooks/11111111",
        "test_url": "https://api.github.com/repos/jerriep/jerriepelser.com/hooks/11111111/test",
        "ping_url": "https://api.github.com/repos/jerriep/jerriepelser.com/hooks/11111111/pings",
        "last_response": {
            "code": 200,
            "status": "active",
            "message": "OK"
        }
    }
]
```

As you can see in the result above, I can easily identify the webhook which was registered by Netlify by the URL being `https://api.netlify.com/hooks/github`. 

Now that I have identified the Netlify webhook, all I need to do is to trigger it, and I can do that by making an HTTP POST request to the URL specified in the `test_url` attribute above.

When I do that, and look at the list of deploys for my website in Netlify, you can see that there are 2 builds for the exact same commit SHA:

![View the deploys in Netlify](/assets/images/2017-10-23-scheduling-hugo-posts-on-netlify/netlify-deploys.png)

The first of those two deploys was created when I made a commit earlier. The second one was triggered by making a request to that `test_url` for the webhook.

So now all I need to do come 6PM, is to do that again and this blog post will be published.

If you are reading this blog post, it means that it worked ;)

## Conclusion

In this blog post I discussed two possibilities for future posting using Hugo and Netlify. Both approaches are acceptable, though the first one (using the pull request) has the advantage that other people can view a preview of the website with the new content before the pull request is merged.

I did not show you actual code of how you would do this, but instead only discussed the possibilities at a higher, more abstract level. Rest assured though that I have tested both approaches and they work. But it is up to you to go and write the code to make this happen!