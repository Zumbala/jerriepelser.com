---
date: 2017-09-05T00:00:00Z
description: |
  The .NET Core CLI allow you to supply custom templates for creating new projects. Here's looking at some resources for developing your own templates
tags:
- .net core
- csharp
title: Tips for developing templates for dotnet new
url: /blog/tips-for-developing-dotnet-new-templates/
---

When creating a new project with the `dotnet new` command,  you are offered a list of templates you can use. These templates allow you to quickly bootstrap a new application according to a specific configuration. There are for example templates to create an MVC application, a Razor application, an API etc.

You can also install extra templates which have been created by other authors. You can discover these templates on the [.NET Templating Wiki](https://github.com/dotnet/templating/wiki/Available-templates-for-dotnet-new) or on the [dotnet core templates website](http://dotnetnew.azurewebsites.net/).

You are also able to develop your own templates, either for internal use, or to make it available on NuGet for other people to take advantage of.

I recently [developed templates](https://github.com/auth0-community/auth0-dotnet-templates) that allow developers to bootstrap a new application that makes use of Auth0 for authentication. 

Here are useful resources I came across, as well as some tips for issues I ran into.

## Useful Resources

* The [dotnet/templating repo](https://github.com/dotnet/templating) is where the source code for the Templating engine lives. It contains a [Wiki](https://github.com/dotnet/templating/wiki) with resources you can explore.

* If you want to see how the standard templates were created, you can find these in the [templating/template_feed folder](https://github.com/dotnet/templating/tree/master/template_feed) of the above mentioned repository.

* Speaking of samples, also check out the [dotnet/dotnet-template-samples repo](https://github.com/dotnet/dotnet-template-samples) which contains samples for a range of different scenarios.

* Next, I suggest you read the [How to create your own templates for dotnet new](https://blogs.msdn.microsoft.com/dotnet/2017/04/02/how-to-create-your-own-templates-for-dotnet-new/) blog post by Sayed Hashimi.

* The .NET Core documentation also contains a useful document on [Custom templates for dotnet new](https://docs.microsoft.com/en-us/dotnet/core/tools/custom-templates) as well as a tutorial entitled [Create a custom template for dotnet new](https://docs.microsoft.com/en-us/dotnet/core/tutorials/create-custom-template).

* Finally there are a couple of videos by Sayed Hashimi on [creating templates for dotnet new](https://www.youtube.com/playlist?list=PLqSOaIdv36hQdvGVkMXDJJ4Uh-nGj3NRK).

## A few issues I ran into

### Identity and Group Identity 

I created 2 templates for Auth0 - one for MVC apps and the other for APIs. I initially thought that the `identity` for each template should be unique, but that both templates should have the same `groupIdentity`. What happened however was that when I installed both templates, only one of them showed up. 

I came across [this issue](https://github.com/dotnet/templating/issues/649) on GitHub which explained that this is by design. This is used when you have the sample template which is available in different language versions.

If you want the templates to show up as separate listings, they should have different group identities.

### Customise parameter names

You can specify extra parameters for your template in the `symbols` section of your `template.json` file, e.g.

```json
{
    "$schema": "http://json.schemastore.org/template",
    [...]
    "symbols": {
        "SignatureAlgorithm": {
            "type": "parameter",
            "datatype": "choice",
            "choices": [
                {
                    "choice": "RS256",
                    "description": "Tokens are signed using RS256"
                },
                {
                    "choice": "HS256",
                    "description": "Tokens are signed using HS256"
                }
            ],
            "description": "The algorithm with which tokens returned from Auth0 are being signed",
            "defaultValue": "RS256"
        },
        "SaveTokens": {
            "type": "parameter",
            "datatype": "bool",
            "description": "If specified, saves the tokens returned from Auth0",
            "defaultValue": "false"
        },
        "IncludeProfilePage": {
            "type": "parameter",
            "datatype": "bool",
            "description": "If specified, adds a page which allows you to view the user's profile",
            "defaultValue": "true"
        },
        "IncludeClaimsPage": {
            "type": "parameter",
            "datatype": "bool",
            "description": "If specified, adds a page which allows you to view the user's claims",
            "defaultValue": "false"
        }
    }
}
```

When you run `dotnet new` for your template with the `-h` option, you will be presented with the list of these options. 

![](/assets/images/2017-09-05-tips-for-developing-dotnet-new-templates/options.png)

However, it did not quite like the fact that the templating engine automatically chose `-S` and `-Sa` as the short options `SignatureAlgorithm` and `SaveTokens` options respectively.

It turns out that you have control over these, by including `dotnetcli.host.json` file in the `.template.config` folder. Inside this file you an specify custom short and long names for each of the options. Here is the example of the one I created:

```json
{
    "$schema": "http://json.schemastore.org/dotnetcli.host",
    "symbolInfo": {
      "SignatureAlgorithm": {
        "longName": "signature-algorithm",
        "shortName": "sa"
      },
      "SaveTokens": {
        "longName": "save-tokens",
        "shortName": "st"
      },
      "IncludeProfilePage": {
        "longName": "include-profile-page",
        "shortName": "pp"
      },
      "IncludeClaimsPage": {
        "longName": "include-claims-page",
        "shortName": "cp"
      }
    }
}
```

With that in place, you can see that the names of the options are presented according to the information in this file:

![](/assets/images/2017-09-05-tips-for-developing-dotnet-new-templates/custom-options.png)

## Conclusion

I am very happy about the inclusion of the new .NET Templating engine. It is still a little limited in some senses, but it is a great step forward. I hope to see the community contribute a lot of new templates.