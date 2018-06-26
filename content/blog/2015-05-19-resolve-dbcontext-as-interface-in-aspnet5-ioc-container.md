---
date: 2015-05-19T00:00:00Z
description: |
  Shows how you can extract an interface from your DbContext and then resolve the interface with the ASP.NET 5 dependency injection framework
tags:
- aspnet
- aspnetmvc
- aspnet5
title: Resolve your DbContext as an interface using the ASP.NET 5 dependency injection framework
url: /blog/resolve-dbcontext-as-interface-in-aspnet5-ioc-container/
---

When developing ASP.NET applications, I prefer to have my database context implement an interface and then inject the interface, as this allows me to more easily mock the database context during unit tests. 

So in a typical application, I would do something like this:

``` csharp
public interface IApplicationDbContext
{
    DbSet<Episode> Episodes { get; set; }
    DbSet<ApplicationUser> Users { get; set; }
    int SaveChanges();
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>, IApplicationDbContext
{
    public virtual DbSet<Episode> Episodes { get; set; }

    public ApplicationDbContext()
    {
		...
    }
}
```

And then inside my controllers, I will inject `IApplicationDbContext`:

```csharp
public class EpisodesController : Controller
{
    private readonly IApplicationDbContext dbContext;

    public EpisodesController(IApplicationDbContext dbContext)
    {
        this.dbContext = dbContext;
    }

	...
}
```

For ASP.NET 4.5 I have been using Autofac, but for ASP.NET 5 I am using the built-in dependency injection mechanisms, so in your typical scenario, the registration of your database context and related services will look like this:

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    ...

    // Add EF services to the services container.
    services.AddEntityFramework()
        .AddSqlServer()
        .AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration["Data:DefaultConnection:ConnectionString"]));

	...
}
```

Thanks to the ASP.NET 5 stack being open source we can [look at the source code](https://github.com/aspnet/EntityFramework/blob/dev/src/EntityFramework.Core/Infrastructure/EntityFrameworkServicesBuilder.cs#L68-L77) for the `AddDbContext` extension method (for beta 4 - maybe this changes in the future):

``` csharp
public virtual EntityFrameworkServicesBuilder AddDbContext<TContext>([CanBeNull] Action<DbContextOptionsBuilder> optionsAction = null)
    where TContext : DbContext
{
    _serviceCollection.AddSingleton(_ => DbContextOptionsFactory<TContext>(optionsAction));
    _serviceCollection.AddSingleton<DbContextOptions>(p => p.GetRequiredService<DbContextOptions<TContext>>());

    _serviceCollection.AddScoped(typeof(TContext), DbContextActivator.CreateInstance<TContext>);

    return this;
}
```

The method (in its current state) does not allow you to pass along separate genric types for the service and implementation, so I thought no problem, I will just add another line to register the service and implementation in the `ConfigureServices` method:

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    ...

    // Add EF services to the services container.
    services.AddEntityFramework()
        .AddSqlServer()
        .AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration["Data:DefaultConnection:ConnectionString"]));

	// Register the service and implementation for the database context
	services.AddScoped<IApplicationDbContext, ApplicationDbContext>();

	...
}
```

This however did not turn out too well:     

![](/assets/images/2015-05-19-resolve-dbcontext-as-interface-in-aspnet5-ioc-container/internal-server-error.png)

This is because the dependency injection framework will just try to create an instance `ApplicationDbContext` without all other configuration settings which is required - which the `AddDbContext` extensions method adds for you.

The solution was simple and that was to supply a lambda expression in the dependency injection registration which resolves an actual instance through the container and will therefore use the correct registration done by the `AddDbContext` method:

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    ...

    // Add EF services to the services container.
    services.AddEntityFramework()
        .AddSqlServer()
        .AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration["Data:DefaultConnection:ConnectionString"]));

	// Register the service and implementation for the database context
	services.AddScoped<IApplicationDbContext>(provider => provider.GetService<ApplicationDbContext>());

	...
}
```
 
This configuration allows me to inject `IApplicationDbContext` into my controllers instead of `ApplicationDbContext`.
