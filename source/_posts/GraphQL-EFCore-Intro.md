---
title: AspNetCore 基于EF Core + HotChocolate 实现的GraphQL查询
date: 2021-07-21 13:01:33
tags:
  - GraphQL
  - EF Core
  - EntityFrameworkCore
  - AspNetCore
---

> GraphQL:  A query language for your API

在开发常见的CRUD应用时，最长遇到的需求是对特定的几个实体进行增删改查，比较常见的方式是手写对应的CRUD逻辑，麻烦且重复，即使引入了EFCore这类ORM，本文介绍如何在常见的CRUD项目中集成GraphQL，减少我们手写重复相关逻辑的麻烦。

本文相关代码都在 https://github.com/Itoktsnhc/demo-graphql  中，为了简单起见，数据库使用SQLite。

## 新建项目 添加依赖
新建Web API项目，添加以下依赖
```
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.8" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="5.0.8">
	<PrivateAssets>all</PrivateAssets>
	<IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
<PackageReference Include="HotChocolate.AspNetCore" Version="11.3.1" />
<PackageReference Include="HotChocolate.Data" Version="11.3.1" />
<PackageReference Include="HotChocolate.Data.EntityFramework" Version="11.3.1" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="5.0.8" />
```

## 数据库模型+ DbContext +初始化配置
``` c#
public class CrudBase //更新时共有字段
{
	public string Name { get; set; }
	public DateTime UpdateTime { get; set; }
}

public class Post: CrudBase
{
	public int Id { get; set; }
	public string Content { get; set; }
	public int BlogId { get; set; }

}

public class Blog: CrudBase
{
	public int Id { get; set; }
	public virtual ICollection<Post> Posts { get; set; }
}

public class DemoDbContext : DbContext
{
	public DemoDbContext(DbContextOptions<DemoDbContext> options) : base(options)
	{

	}

	public DbSet<Post> Posts { get; set; }
	public DbSet<Blog> Blogs { get; set; }
}
```

配置PooledDbContextFactory
``` c#
public void ConfigureServices(IServiceCollection services)
{
	// 注册PooledDbContext
	services.AddPooledDbContextFactory<DemoDbContext>((_, op) => op.UseSqlite("Data Source=./demo.db"));
	// ...
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
	//启动时做迁移
	using (var scope = app.ApplicationServices.GetRequiredService<IServiceScopeFactory>().CreateScope())
	{
		scope
			.ServiceProvider
			.GetRequiredService<IDbContextFactory<DemoDbContext>>()
			.CreateDbContext().Database.Migrate();
	}
	//...
}
```

添加migration
```
dotnet ef migrations add Init;
```


## GraphQL
本文涉及到两个类型: Query, Mutation. 简单理解Query面向查询，Mutation面向增删改
### Query: 
```c#
public class Query
{
	[UseDbContext(typeof(DemoDbContext))]
	[UseOffsetPaging(DefaultPageSize = 10, MaxPageSize = 50, IncludeTotalCount = true)]
	[UseProjection]
	[UseFiltering]
	[UseSorting]
	public IQueryable<Blog> Blogs([ScopedService] DemoDbContext context)
	{
		return context.Blogs.AsNoTracking();
	}

	[UseDbContext(typeof(DemoDbContext))]
	[UseOffsetPaging(DefaultPageSize = 10, MaxPageSize = 50, IncludeTotalCount = true)]
	[UseProjection]
	[UseFiltering]
	[UseSorting]
	public IQueryable<Post> Posts([ScopedService] DemoDbContext context)
	{
		return context.Posts.AsNoTracking();
	}
}
```
可以看到使用了一些数据注解，就能在基于`IQueryable<T>`的基础上实现翻页，排序，过滤等功能

### Mutation:
```c#
public class Mutation
{
	[UseDbContext(typeof(DemoDbContext))]
	public async Task<Post> AddPostAsync([ScopedService] DemoDbContext context, Post entity)
	{
		return await AddEntityAsync(context, entity);
	}

	[UseDbContext(typeof(DemoDbContext))]
	public async Task<Post> DeletePostAsync([ScopedService] DemoDbContext context, int postId)
	{
		return await DeleteEntityAsync<Post>(context, postId);
	}

	[UseDbContext(typeof(DemoDbContext))]
	public async Task<Post> UpdatePostAsync([ScopedService] DemoDbContext context, int postId, Post post)
	{
		return await UpdateEntityAsync<Post>(context, postId, post);
	}

	private static async Task<T> AddEntityAsync<T>(DemoDbContext context, T entity) where T : CrudBase
	{
		entity.UpdateTime = DateTime.UtcNow;
		if (await context.Set<T>().AnyAsync(
			e => e.Name == entity.Name))
		{
			throw new Exception($"{typeof(T).Name} with name exists:{entity.Name}");
		}

		context.Set<T>().Add(entity);
		await context.SaveChangesAsync();
		return entity;
	}

	private static async Task<T> DeleteEntityAsync<T>(DemoDbContext context, int id) where T : CrudBase
	{
		var entity = await context.Set<T>().FirstOrDefaultAsync(e => e.Id == id);
		if (entity != null)
		{
			context.Set<T>().Remove(entity);
			await context.SaveChangesAsync();
		}
		else
		{
			throw new Exception("cannot delete not exist entity");
		}

		return entity;
	}

	private static async Task<T> UpdateEntityAsync<T>(DemoDbContext context, int id, T entity) where T : CrudBase
	{
		entity.Id = id;
		if (!await context.Set<T>().AnyAsync(e => e.Id == entity.Id))
		{
			throw new Exception("cannot update not exist entity");
		}
		if (await context.Set<T>().AnyAsync(e => e.Name == entity.Name && e.Id != entity.Id))
		{
			throw new Exception($"{typeof(T).Name} with same name exists:{entity.Name}");
		}
		entity.UpdateTime = DateTime.UtcNow;
		context.Entry(entity).State = EntityState.Modified;
		await context.SaveChangesAsync();
		return entity;
	}
}
```

### 配置GraphQL
``` c#
//注册服务
            services.AddGraphQLServer()
                .AddQueryType<Query>()
                .AddMutationType<Mutations>()
                .AddProjections()
                .AddFiltering()
                .AddSorting()
                .AddErrorFilter<GraphQLErrorFilter>();

//
app.UseEndpoints(endpoints =>
{
	endpoints.MapControllers();
	endpoints.MapGraphQL();
});
```

### 测试
启动程序后浏览器打开 `{host}/graphql`
``` graphql
//新增Blog
mutation {
  addBlog(blog: { id: 0, name: "123", updateTime: "2021.07.21 10:55:55" }) {
    id
    name
  }
}


//新增post
mutation {
  addPost(
    entity: {
      id: 0
      name: "post2"
      content: "post1 content"
      updateTime: "2021.07.21 10:55:55"
    }
  ) {
    id
    name
  }
}

//查询Post
query {
  posts {
    items {
      id
      name
      content
    }
    totalCount
  }
}

// 查询Blog关联的Post
query {
  blogs {
    items {
      id
      name
      posts {
        id
        name
        content
      }
    }
    totalCount
  }
}

```
## 相关资源:
- https://github.com/ChilliCream/hotchocolate
- https://docs.microsoft.com/en-us/ef/core
- https://graphql.org/
