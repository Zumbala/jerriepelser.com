---
title: "Using ShareX for blogging or documentation screenshots"
description: |
  A walthrough of my ShareX configuration for creating screenshots for blog posts and other forms of documentation.
date: 2018-01-25
tags:
- hugo
- netlify
- sharex
url: /blog/using-sharex-for-blogging-screenshots
---

As mentioned on [a previous occasion](/blog/why-i-like-netlify), my blog uses [Hugo](https://gohugo.io/) and is hosted on [Netlify](https://www.netlify.com/). I use [ShareX](https://getsharex.com/) for taking screenshots, so today I want to talk a bit about the workflow I use to create screenshots for my blog posts.

## My blog folder structure

Before we carry on, I would like to discuss the folder structure for my Hugo repository to give some context. The two main folders of interest are the `/content/blog` folder and the `/static/assets/images` folders:

![Hugo website strucure](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/hugo-folder-strucure.png)

The `/content/blog` folder contains all of the blog posts:

![Blog folder](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/blog-folder.png)

The `/static/assets/images` folder contains all of the images for the blog posts. Each blog post will typically have its own folder with a list of images, for example:

![Images folder](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/images-folder.png)

As a further example, the files relevant to this particular blog post are laid out as follows:

```text
- content
    - blog
        - 2018-01-25-using-sharex-for-blogging-screenshots.md
- assets
    - images
        - 2018-01-25-using-sharex-for-blogging-screenshots
            - ...all images for this blog
```

So the markdown file for this blog post is located at `/content/blog/2018-01-25-using-sharex-for-blogging-screenshots.md` and all the images are located in the folder `/assets/images/2018-01-25-using-sharex-for-blogging-screenshots`.

## My blogging workflow

When writing a blog post, I will typically already have a piece of code somewhere which demonstrates (at least in broad terms) what I want to blog about, since I usually blog about issues I come across during my daily software development.

I will however typically create a new GitHub repository with a project which demonstrates whatever is being discussed in the blog post. I will write the blog post and the source code in tandem, so both progress at the same time.

While doing this, I will also take a bunch of screenshots to demonstrate certain things. These screenshots will typically be of the Visual Studio IDE, the web browser, the command line prompt, etc. 

I also take these screenshots as I progress with writing the blog post and the source code, as I find this the least interrupting to my "flow".

Until I started looking at optimising this screenshot process, what would happen was that I would take a screenshot in ShareX which would then save it to a sub-folder in my Documents folder. It would use random names for these screenshots, e.g.

![ShareX screenshot folder](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/sharex-screenshot-folder.png)

When I took a screenshot, I would place an empty markdown image tag, i.e. `![]()`, in my markdown document as a placeholder so I can remember to update it with the correct path to the image.

Once I am done with the blog post, I would then go back to the images folder. Some images would require annotation, for example, to highlight a certain area. I would do those and then give all the images proper, descriptive names - both for own sanity, but also to help SEO.

I would then upload the images to TinyPNG to compress them, and download the compressed images. Once done I would copy them all to the correct folder under the `/static/assets/images` folder. Finally, I would go through the markdown document and update all those empty image placeholders with the correct image path.

## The enhanced workflow

So, I set about enhancing this workflow with ShareX. First thing I did was go to the ShareX Hotkey Settings, and clicked the gear icon next to the **Ctrl + Print Screen** shortcut:

![ShareX Hotkey Settings](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/sharex-hotkey-settings.png)

I enabled the **Override "After Capture" settings**, and in the dropdown I chose the **Open in image editor**, **Save image to file as...** and **Copy file path to clipboard** options.

![ShareX Capture Settings](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/sharex-capture-settings.png)

With that simple change, I have now significantly enhanced my workflow. Now, when I want to take a screenshot I simply press the **Ctrl + Print Screen** key combination, which will invoke the ShareX screen capture. I will select the area I want to capture, and ShareX will open the Image Editor:

![ShareX Image Editor](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/sharex-image-editor.png)

I then make the necessary annotations, if required, and once done click that green check mark icon, or just press Enter. This will then open a Save File dialog, and I will give the file a proper name and save it directly to the blog post's images folder under `/static/assets/images`:

![Save the image](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/save-image.png)

> The first time I save a file for a blog post, I will need to create and go to the correct folder under `/static/assets/images`, but after that the Save File dialog will remember that last folder. So on every subsequent file save action, the dialog is already at the correct folder and all I need to do is give it a correct name.

After the file has been saved, ShareX will copy the file location to the clipboard. I create an image tag in my markdown document and copy the path from the clipboard:

```markdown
![Save the image](C:\Development\jerriep\jerriepelser.com\static\assets\images\2018-01-25-using-sharex-for-blogging-screenshots\save-image.png)
```

As you can see, the file path will be the full file location, so I just need to quickly fix it up properly:

```markdown
![Save the image](/assets/images/2018-01-25-using-sharex-for-blogging-screenshots/save-image.png)
```

With that, I now have streamlined my screenshot process to do everything at once while writing the blog post. No more having to fix filenames, copy images to the correct location and update the image tags afterwards.

You may, however, have noticed that I have missed the image compression step I did previously with TinyPNG. Well, I have blogged previously about [compressing images using the TinyPNG CLI](/blog/compress-images-using-tinypng-cli/), so that is the final step. I go to the correct folder where the images for the blog post is stored and compress them with the TinyPNG CLI. This takes mere seconds and because the images are compressed inline, there is no uploading image to and downloading images from the TinyPNG website involved.

**Edit:** I just discovered this [TinyPNG extension for VS Code](https://marketplace.visualstudio.com/items?itemName=andi1984.tinypng) which allows me to compress the images in the folder directly from VS Code without having to switch to the command prompt.

## Conclusion

In this blog post, I showed how a simple workflow change made my blogging and screenshot-ing experience much more streamlined. If you blog or write docs regularly, then hopefully this can help you.
