---
title: "Store a Dictionary as a JSON string using EF Core 2.1"
description:
  Demonstrates how you can store the contents of a Dictionary property as a JSON document in your database when using EF Core 2.1
tags:
- ef core
- .net core
---

I have an entity in my Entity Framework model which has a property of type `Dictionary<string, string>` which I would like to store as a JSON string in a `nvarchar` field in my database.

The definition of the entity is as follows:

```csharp
public class PublishSource
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required]
    public string Name { get; set; }

    [Required]
    public Dictionary<string, string> Properties { get; set; } = new Dictionary<string, string>();
}
```

Converting this to and from a JSON document which can be stored in the database turns out to be pretty easy with the [Value Conversions](https://docs.microsoft.com/en-us/ef/core/modeling/value-conversions) which were added in EF Core 2.1.

In the `OnModelCreating` method of the database context I just call `HasConversion`, which does the serialization and deserialization of the dictionary:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<PublishSource>()
        .Property(b => b.Properties)
        .HasConversion(
            v => JsonConvert.SerializeObject(v),
            v => JsonConvert.DeserializeObject<Dictionary<string, string>>(v));
}
```

**One important thing** I have noticed, however, is that when updating the entity and changing items in the dictionary, the EF change tracking does not pick up on the fact that the dictionary was updated, so you will need to explicitly call the `Update` method on the `DbSet<>` to set the entity to modified in the change tracker.