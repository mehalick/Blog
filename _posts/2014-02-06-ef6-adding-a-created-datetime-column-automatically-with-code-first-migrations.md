---
layout: post
title: "EF6: Adding a Created Date/Time Column Automatically with Code First Migrations"
date: 2014-02-06
post_id: 8
---

Here’s a quick one, I’m using EF6 code first migrations and I want to add a created date/time column to every table in the database. I want it to have a default value of GETUTCDATE() and ultimately look like:

![](https://dl.dropboxusercontent.com/u/38696855/blog/8/2-6-2014%2012-07-35%20PM.png)

I’m using UTC dates so I don’t have to worry about the time zone of the server where this ultimately lives and calling the column **CreatedUtc** because it’s concise but clear.

So let’s get started, I’ll create a new class library in Visual Studio called CreatedUtcColumnDemo and immediately add Entity Framework via Nuget.

<pre class="cmd">PM> Install-Package EntityFramework</pre>

Now we’ll add our first entity and data context, notice the **CreatedUtc** property with data annotations:

{% highlight csharp %}
namespace CreatedUtcColumnDemo
{
    public class Product
    {
        public int ProductId { get; set; }
        public string Name { get; set; }

        [Required, DatabaseGenerated(DatabaseGeneratedOption.Computed)]
        public DateTime CreatedUtc { get; set; }
    }

    public class DataContext : DbContext
    {
        public DbSet<product> Products { get; set; }
    }
}
{% endhighlight %}

Because we’ll ultimately want our **CreatedUtc** property on all entities let’s extract an abstract base class (EntityBase) and have Product inherit from it:

{% highlight csharp %}
namespace CreatedUtcColumnDemo
{
    public class Product : EntityBase
    {
        public int ProductId { get; set; }
        public string Name { get; set; }  
    }

    public abstract class EntityBase
    {
        [Required, DatabaseGenerated(DatabaseGeneratedOption.Computed)]
        public DateTime CreatedUtc { get; set; }
    }

    public class DataContext : DbContext
    {
        public DbSet<product> Products { get; set; }
    }
}
{% endhighlight %}

Now we’ll enable migrations and create a single migration to create our Products table:

<pre class="cmd">PM> Enable-Migrations</pre>

<pre class="cmd">PM> Add-Migration AddProduct</pre>

At this point our solution looks like:

![](https://dl.dropboxusercontent.com/u/38696855/blog/8/2-6-2014%2011-52-49%20AM.png)

We now want to ensure that when we update our database via code first migrations  our **CreatedUtc** property gets a default value, in this case GETUTCDATE(). One option would be to adjust the migration class manually, for each entity we _could_ update CreateTable(), notice the addition of defaultValueSql: "GETUTCDATE()".

{% highlight csharp %}    
public partial class AddProduct : DbMigration
{
    public override void Up()
    {
        CreateTable(
            "dbo.Products",
            c => new
                {
                    ProductId = c.Int(nullable: false, identity: true),
                    Name = c.String(),
                    CreatedUtc = c.DateTime(nullable: false, defaultValueSql: "GETUTCDATE()"),
                })
            .PrimaryKey(t => t.ProductId);

    }

    public override void Down()
    {
        DropTable("dbo.Products");
    }
}
{% endhighlight %}

This would become tedious and error prone as we start to add new entities though. A more efficient approach would be to create a custom SqlServerMigrationSqlGenerator and update our migration configuration to use this. Jumping right to the code:

{% highlight csharp %}
namespace CreatedUtcColumnDemo.Migrations
{
    internal sealed class Configuration : DbMigrationsConfiguration<DataContext>
    {
        public Configuration()
        {
            AutomaticMigrationsEnabled = false;
            SetSqlGenerator("System.Data.SqlClient", new CustomSqlServerMigrationSqlGenerator());
        }
    }

    internal class CustomSqlServerMigrationSqlGenerator : SqlServerMigrationSqlGenerator
    {
        protected override void Generate(AddColumnOperation addColumnOperation)
        {
            SetCreatedUtcColumn(addColumnOperation.Column);

            base.Generate(addColumnOperation);
        }

        protected override void Generate(CreateTableOperation createTableOperation)
        {
            SetCreatedUtcColumn(createTableOperation.Columns);

            base.Generate(createTableOperation);
        }

        private static void SetCreatedUtcColumn(IEnumerable<ColumnModel> columns)
        {
            foreach (var columnModel in columns)
            {
                SetCreatedUtcColumn(columnModel);
            }
        }

        private static void SetCreatedUtcColumn(PropertyModel column)
        {
            if (column.Name == "CreatedUtc")
            {
                column.DefaultValueSql = "GETUTCDATE()";
            }
        }
    }
}
{% endhighlight %}

Notice how CustomSqlServerMigrationSqlGenerator inherits from SqlServerMigrationSqlGenerator and overrides two Generate(…) methods. We then set our custom generator to the default for our project via SetSqlGenerator("System.Data.SqlClient", new CustomSqlServerMigrationSqlGenerator()).

Lastly, we’ll update our local database and we’re done.

<pre class="cmd">PM> Update-Database</pre>

Now, every time we create a new entity we inherit from EntityBase, when we update our database with migrations it will always add a **CreatedUtc** column with a default value of GETUTCDATE().

You can find the demo source code at [https://github.com/andy-mehalick/CreatedUtcColumnDemo](https://github.com/andy-mehalick/CreatedUtcColumnDemo "https://github.com/andy-mehalick/CreatedUtcColumnDemo").