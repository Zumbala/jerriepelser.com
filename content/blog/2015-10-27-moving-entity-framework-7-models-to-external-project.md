---
date: 2015-10-27T00:00:00Z
description: |
  A look at how you can keep your Entity Framework models in a separate project from your main ASP.NET 5 project.
tags:
- asp.net 5
- entity framework
- entity framework 7
title: Moving your Entity Framework 7 models to an external project
url: /blog/moving-entity-framework-7-models-to-external-project/
---

## Introduction

In most of my projects - except perhaps for the most basic ones - I like to keep my Entity Framework data context and models in a separate project from my main ASP.NET application, so I can easily share it amongst other projects in the solution.

I had to figure out how to do this in a new ASP.NET 5 project recently, and could not find any sort of guidance for this available on the web. The error messages for the DNX Entity Framework tooling was good enough to guide me along the way, but here are the steps for you if you want to accomplish the same thing so you do not have to spend time figuring this out.

**FYI: This blog post was done with Beta 8.**

## Put Entity Framework models in a separate project

First off create a new project in Visual Studio and select the standard ASP.NET 5 web application with Authentication:

![](/assets/images/moving-entity-framework-7-models-to-external-project/new-project-dialog.png)

This will leave you with a project structure which looks more or less than this:

![](/assets/images/moving-entity-framework-7-models-to-external-project/project-structure.png)

Next, delete the Migrations folder, as these will need to be re-created in the new project in any case:

![](/assets/images/moving-entity-framework-7-models-to-external-project/delete-migrations.png)

Now add a new project, and under the Web folder select the "Class Library (Package)" project type. I called it ExternalEntityFramework.Data, but you can call it whatever makes sense to you:  

![](/assets/images/moving-entity-framework-7-models-to-external-project/create-class-library.png)

Once the project has been created, you can delete the Class1.cs file.

In your ASP.NET 5 application, add a reference to the new project:

![](/assets/images/moving-entity-framework-7-models-to-external-project/add-reference.png)

Now head back to the new project and open the project.json file and add the following dependencies:

``` json
"dependencies": {
"EntityFramework.Commands": "7.0.0-beta8",
"EntityFramework.SqlServer": "7.0.0-beta8",
"Microsoft.AspNet.Identity.EntityFramework": "3.0.0-beta8",
}
```

Also add a commands section with the `ef` command, because you will need that to execute the commands for Entity Migrations:

``` json
"commands": {
"ef": "EntityFramework.Commands"
}
``` 

Here is my complete project.json after the changes:

``` json
{
  "version": "1.0.0-*",
  "description": "ExternalEntityFramework.Data Class Library",
  "authors": [ "jerriep" ],
  "tags": [ "" ],
  "projectUrl": "",
  "licenseUrl": "",

  "dependencies": {
    "EntityFramework.Commands": "7.0.0-beta8",
    "EntityFramework.SqlServer": "7.0.0-beta8",
    "Microsoft.AspNet.Identity.EntityFramework": "3.0.0-beta8",
  },

  "commands": {
    "ef": "EntityFramework.Commands"
  },

  "frameworks": {
    "dnx451": { },
    "dnxcore50": {
      "dependencies": {
        "Microsoft.CSharp": "4.0.1-beta-23409",
        "System.Collections": "4.0.11-beta-23409",
        "System.Linq": "4.0.1-beta-23409",
        "System.Runtime": "4.0.21-beta-23409",
        "System.Threading": "4.0.11-beta-23409"
      }
    }
  }
}
``` 

Now move the `ApplicationDbContext.cs` and `ApplicationUser.cs` files from your ASP.NET 5 project to the new project:

![](/assets/images/moving-entity-framework-7-models-to-external-project/move-models.png)

Fix the namespaces in your `ApplicationDbContext` and `ApplicationUser` classes. In my case I changed it from `ExternalEntityFramework.Models` to `ExternalEntityFramework.Data`. 

Make sure that you fix the namespace references in your ASP.NET 5 application in the following files:

* Startup.cs
* Controllers/AccountController.cs
* Controllers/ManageController.cs
* Views/_ViewImports.cs

## Enable Entity Framework migrations

At this point you should be able to build the project without any errors. 

We still need to enable you to work correctly with Entity Framework migrations. If you are going to try and add a migration you will get some errors:

![](/assets/images/moving-entity-framework-7-models-to-external-project/dnx-ef-errors.png)

So it seems we need a Startup class with some configuration of the Entity Framework. Right click on the project and select Add > New Item. In the dialog select the "Startup class" template:

![](/assets/images/moving-entity-framework-7-models-to-external-project/add-startup-class.png)

The class needs some basics to configure the Entity Framework correctly. This is what mine looks like:

``` csharp
public class Startup
{
    public IConfigurationRoot Configuration { get; set; }

    public Startup(IHostingEnvironment env, IApplicationEnvironment appEnv)
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(appEnv.ApplicationBasePath)
            .AddJsonFile("appsettings.json");

        Configuration = builder.Build();
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddEntityFramework()
            .AddSqlServer()
            .AddDbContext<ApplicationDbContext>(options =>
                options.UseSqlServer(Configuration["Data:DefaultConnection:ConnectionString"]));
    }
    public void Configure(IApplicationBuilder app)
    {
    }
}
```

You will notice in the `Startup` class we reference an `appsettings.json` file which contains the connection string for the database, so we need to add this file. There is already an `appsettings.json` file in your ASP.NET application, so just copy that over to the project containing your Entity Framework classes.

Now I can run the `migrations` command again successfully:

![](/assets/images/moving-entity-framework-7-models-to-external-project/dnx-ef-migrations-add.png)

This will add the correct migrations to my project:

![](/assets/images/moving-entity-framework-7-models-to-external-project/project-structure-with-migrations.png)

And you can also execute the migrations successfully:

![](/assets/images/moving-entity-framework-7-models-to-external-project/execute-migrations.png)

## Conclusion

In this blog post I showed you how you can move your Entity Framework models to an external project. This allows you to more easily share those models among other projects in your solution.