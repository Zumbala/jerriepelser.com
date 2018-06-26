---
title: Using MariaDB with ASP.NET Core 2.0
description: |
  MariaDB is an open source database compatible with MySQL. Here's looking at how you can use this in your ASP.NET Core application.
date: 2017-12-12
tags:
- .net core
- asp.net
- asp.net core
- ef core
- mariadb
url: /blog/using-mariadb-with-aspnet-core
---

As part of my recent explorations I have looked into various ways of hosting an ASP.NET Core application. One path I explored was using MariaDB as an alternative to the SQL Server world which most .NET developers are used to.

**So what is MariaDB?** From the [Wikipedia article about it](https://en.wikipedia.org/wiki/MariaDB):

> MariaDB is a community-developed fork of the MySQL relational database management system intended to remain free under the GNU GPL. Development is led by some of the original developers of MySQL, who forked it due to concerns over its acquisition by Oracle Corporation. Contributors are required to share their copyright with the MariaDB Foundation.
>
> MariaDB intends to maintain high compatibility with MySQL, ensuring a drop-in replacement capability with library binary equivalency and exact matching with MySQL APIs and commands.

So basically it is a fork of MySQL which is guaranteed to stay open source. And as noted it is supposed to be a drop-in replacement for MySQL. 

So let's put this to the test with a simple ASP.NET Core application.

## Installing MariaDB

You can download MariaDB by heading to the [downloads section](https://downloads.mariadb.org/) of the MariaDB Foundation website. Once there go ahead and download the correct version for your platform. For this blog post I used **version 10.2.11** on **Windows**.

![The available MariaDB downloads for Windows](/assets/images/2017-12-12-using-mariadb-with-aspnet-core/mariadb-downloads.png)

Once the download has completed you can proceed through the installation process. At some point the installer will prompt you for a **root** user password. Make a note of that as we'll be using that password in the connection string from the application.

![Specifying a root user password during installation](/assets/images/2017-12-12-using-mariadb-with-aspnet-core/root-password.png)

## Creating your ASP.NET Core application

For the ASP.NET Core application we'll create a normal MVC application with individual user accounts:

```bash
dotnet new mvc --name MariaDBTest --auth Individual
```

Since MariaDB claims binary compatibility with MySQL we can use a MySQL Entity Framework Core provider. In this case I used [`Pomelo.EntityFrameworkCore.MySql`](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql) (version 2.0.1) since that appears one of the best maintained packages. 

Add a reference to the package:

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
```

The `mvc` template I used has configured the application to use Sqlite. So currently in the `Startup` class, this is what the `ConfigureServices` method looks like:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlite(Configuration.GetConnectionString("DefaultConnection")));

    // code omitted for brevity...
}
```

Let's update that to use MySQL:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseMySql(Configuration.GetConnectionString("DefaultConnection")));

    // code omitted for brevity...
}
```

And of course we need to update the `DefaultConnection` in the `appsettings.json` file to the correct MariaDB connection string. Specify the **User Id** as **root** and password which you supplied during installation. In the sample below I specified the name for the database as **mariadbtest** but you can call it something different if you want.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;User Id=root;Password=password;Database=mariadbtest"
  },
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

## Create the EF models

Everything is now configured correctly in our application, so let's create the database migration scripts:

```bash
dotnet ef migrations add CreateIdentityModels
```

And apply the migation to the database:

```bash
dotnet ef database update
```

## Viewing the database

During the MariaDB installation, an application called [HeidiSQL](https://www.heidisql.com/) was installed which allow you to manage MySQL databases (and apparently also SQL Server and PostgreSQL). If it was not installed for you, then [download](https://www.heidisql.com/download.php) and install it. 

Open HeidiSQL and create a new session, specifying the settings for your MariaDB server and the database you created:

![Create a new HeidiSQL session](/assets/images/2017-12-12-using-mariadb-with-aspnet-core/heidisql-create-session.png)

Open the new session and you will see all the tables for ASP.NET Identity was created correctly:

![Viewing the ASP.NET Identity tables in HeidiSQL](/assets/images/2017-12-12-using-mariadb-with-aspnet-core/heidisql-tables.png)

And sure enough, if I run the application and register a new user, you can see the user being added to the **aspnetusers** table:

![Viewing the registered users in HeidiSQL](/assets/images/2017-12-12-using-mariadb-with-aspnet-core/heidisql-users.png)

## Conclusion

In this blog post I demonstrated how easy it is to use a MariaDB database with an ASP.NET Core application. 

I have not used this extensively in production, so I cannot really testify to the viability of this beyond a simple test application. If you want to go this route then validate for yourself whether it will work for your scenario.

Oh and I am not MariaDB/MySQL expert, but logic tells me using the root user for your database **is probably not a good idea**. Please research the correct security best practices and make sure you apply them.