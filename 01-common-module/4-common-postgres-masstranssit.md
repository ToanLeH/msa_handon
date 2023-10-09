# Create MSA.Common.PostgresMassTransit for integrating with Postgres and MassTransit shared across service

# Create MSA.Common.PostgresMassTransit class library project
## Create new class library project
- via Visual studio code, run below command to create new class library project
``` bash
dotnet new classlib -n MSA.Common.PostgresMassTransit
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.Common.PostgresMassTransit
```

## Add reference to MSA.Common.Contracts project
- via Visual studio code, run below command to add reference to MSA.Common.Contracts
``` bash
dotnet add reference ../MSA.Common.Contracts
```

## Add postgres nuget package
- via Visual studio code, run below command to install common nuget package 
``` bash
dotnet add package MassTransit.AspNetCore
dotnet add package MassTransit.EntityFrameworkCore
dotnet add package MassTransit.RabbitMQ
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

## Create folder PostgresDB via MSA.Common.PostgresMassTransit project
- via visual studio code, create folder **PostgresDB**

### Create new file **AppDbContextBase.cs** via PostgresDB folder
- via visual studio code, create file **AppDbContextBase.cs** via PostgresDB folder, paste below code to it

``` C#
using System;
using MassTransit;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using MSA.Common.Contracts.Settings;

namespace MSA.Common.PostgresMassTransit.PostgresDB;

public class AppDbContextBase : DbContext
{
    private readonly IConfiguration configuration;
    private readonly DbContextOptions options;

    public AppDbContextBase(
        IConfiguration configuration,
        DbContextOptions options) 
        : base(options)
    {
        this.configuration = configuration;
        this.options = options;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var serviceSetting = configuration
            .GetSection(nameof(ServiceSetting))
            .Get<ServiceSetting>();

        modelBuilder.HasDefaultSchema(serviceSetting.ServiceName);
        
        base.OnModelCreating(modelBuilder);

        modelBuilder.AddInboxStateEntity(i => {
            i.ToTable("InboxState");
        });
        modelBuilder.AddOutboxMessageEntity(o => {
            o.ToTable("OutboxMessage");
        });
        modelBuilder.AddOutboxStateEntity(o => {
            o.ToTable("OutboxState");
        });
    }
}
```
### Create new file **Extensions.cs** via PostgresDB folder
- via visual studio code, create file **Extensions.cs** via PostgresDB folder, paste below code to it

``` C#
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MSA.Common.Contracts.Domain;
using MSA.Common.Contracts.Settings;

namespace MSA.Common.PostgresMassTransit.PostgresDB;

public static class Extensions
{
    public static IServiceCollection AddPostgres<TDbContext>(this IServiceCollection services)
        where TDbContext : DbContext
    {
        var srvProvider = services.BuildServiceProvider();
        var config = srvProvider.GetService<IConfiguration>();
        var pgSettings = config.GetSection(nameof(PostgresDBSetting)).Get<PostgresDBSetting>();

        services
            .AddEntityFrameworkNpgsql()
            .AddDbContext<TDbContext>(opt => {
                opt.UseNpgsql(pgSettings.ConnectionString, pgOpt => {
                    pgOpt.MigrationsAssembly(typeof(TDbContext).Assembly.GetName().Name);
                });
            });

        return services;
    }

    public static IServiceCollection AddPostgresRepositories<TDbContext, TEntity>(
        this IServiceCollection services) 
        where TDbContext : AppDbContextBase
        where TEntity : class, IEntity 
    {
        // services.AddSingleton<IRepository<TEntity>>(srvProvider => {
        //     var dbContext = srvProvider.GetRequiredService<TDbContext>();
        //     return new PostgresRepository<TDbContext, TEntity>(dbContext);
        // });
        services.AddScoped<IRepository<TEntity>, PostgresRepository<TDbContext,TEntity>>();
        return services;
    }

        public static IServiceCollection AddPostgresUnitofWork<TDbContext>(
        this IServiceCollection services) 
        where TDbContext : AppDbContextBase
    {
        services.AddScoped<PostgresUnitOfWork<TDbContext>, PostgresUnitOfWork<TDbContext>>();
        return services;
    }
}
```

### Create new file **PostgresRepository.cs** via PostgresDB folder
- via visual studio code, create file **PostgresRepository.cs** via PostgresDB folder, paste below code to it

``` C#
using System.Security.Cryptography.X509Certificates;
using System.Linq.Expressions;
using Microsoft.EntityFrameworkCore;
using MSA.Common.Contracts.Domain;

namespace MSA.Common.PostgresMassTransit.PostgresDB;

public class PostgresRepository<TDbContext, TEntity> : IRepository<TEntity>
        where TDbContext : AppDbContextBase
        where TEntity : class, IEntity
{
    private readonly TDbContext _dbcontext;
    private readonly DbSet<TEntity> _dbSet;

    public PostgresRepository(TDbContext dbcontext)
    {
        this._dbcontext = dbcontext;
        this._dbSet = dbcontext.Set<TEntity>();
    }

    public async Task CreateAsync(TEntity entity)
    {
        await _dbSet.AddAsync(entity);
    }

    public async Task DeleteAsync(TEntity entity)
    {
        await Task.FromResult(_dbSet.Remove(entity));
    }

    public async Task<IReadOnlyCollection<TEntity>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<IReadOnlyCollection<TEntity>> GetAllAsync(Expression<Func<TEntity, bool>> filter)
    {
        return await _dbSet.Where(filter).ToListAsync();
    }

    public async Task<TEntity> GetAsync(Guid id)
    {
        return await _dbSet.Where(x => x.Id == id).FirstOrDefaultAsync();
    }

    public async Task<TEntity> GetAsync(Expression<Func<TEntity, bool>> filter)
    {
        return await _dbSet.Where(filter).FirstOrDefaultAsync();
    }

    public async Task UpdateAsync(TEntity entity)
    {
        await Task.FromResult(_dbSet.Update(entity));
    }
}
```

### Create new file **PostgresUnitOfWork.cs** via PostgresDB folder
- via visual studio code, create file **PostgresUnitOfWork.cs** via PostgresDB folder, paste below code to it

``` C#
namespace MSA.Common.PostgresMassTransit.PostgresDB;

public class PostgresUnitOfWork<TDbContext> 
    where TDbContext : AppDbContextBase
{
    private readonly TDbContext dbcontext;

    public PostgresUnitOfWork(TDbContext dbcontext)
    {
        this.dbcontext = dbcontext;
    }

    public async Task SaveChangeAsync() => await dbcontext.SaveChangesAsync();
}
```

## Create folder MassTransit via MSA.Common.PostgresMassTransit project
- via visual studio code, create folder **MassTransit**

### Create new file **Extensions.cs** via PostgresDB folder
- via visual studio code, create file **Extensions.cs** via MassTransit folder, paste below code to it

``` C#
using System.Reflection;
using MassTransit;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.Common.Contracts.Settings;

namespace MSA.Common.PostgresMassTransit.MassTransit;

public static class Extensions
{
    public static IServiceCollection AddMassTransitWithRabbitMQ(this IServiceCollection services)
    {
        services.AddMassTransit(configure => {
            configure.AddConsumers(Assembly.GetEntryAssembly());

            configure.UsingRabbitMq((context, configurator) => {
                var configuration = context.GetService<IConfiguration>();
                var serviceSetting = configuration.GetSection(nameof(ServiceSetting)).Get<ServiceSetting>();
                RabbitMQSetting rabitMQSetting = configuration.GetSection(nameof(RabbitMQSetting)).Get<RabbitMQSetting>();
                configurator.Host(rabitMQSetting.Host);
                configurator.ConfigureEndpoints(context, 
                    new KebabCaseEndpointNameFormatter(serviceSetting.ServiceName, false));
                configurator.UseMessageRetry(retryPoilicy => {
                    retryPoilicy.Interval(3, TimeSpan.FromSeconds(10));
                });
            });
        });

        return services;
    }

    public static IServiceCollection AddMassTransitWithPostgresOutbox<TDbContext>(
        this IServiceCollection services,
        Action<IBusRegistrationConfigurator> furtherConfig = null)
        where TDbContext : AppDbContextBase
    {
        services.AddMassTransit(configure => {
            configure.AddConsumers(Assembly.GetEntryAssembly());

            configure.UsingRabbitMq((context, configurator) => {
                var configuration = context.GetService<IConfiguration>();
                var serviceSetting = configuration.GetSection(nameof(ServiceSetting)).Get<ServiceSetting>();
                RabbitMQSetting rabitMQSetting = configuration.GetSection(nameof(RabbitMQSetting)).Get<RabbitMQSetting>();
                configurator.Host(rabitMQSetting.Host);
                configurator.ConfigureEndpoints(context, 
                    new KebabCaseEndpointNameFormatter(serviceSetting.ServiceName, false));
                configurator.UseMessageRetry(retryPoilicy => {
                    retryPoilicy.Interval(3, TimeSpan.FromSeconds(10));
                });
            });

            configure.AddEntityFrameworkOutbox<TDbContext>(o => {
                o.UsePostgres();
                o.UseBusOutbox();
            });

            furtherConfig?.Invoke(configure);
        });

        return services;
    }
}
```

## Modify **MSA.Common.PostgresMassTransit.csproj** file
- via visual studio code, modify **MSA.Common.PostgresMassTransit.csproj** file as below
``` Diff
  <ItemGroup>
    <PackageReference Include="MassTransit.AspNetCore" Version="7.3.1" />
    <PackageReference Include="MassTransit.EntityFrameworkCore" Version="8.0.16" />
    <PackageReference Include="MassTransit.RabbitMQ" Version="8.0.16" />
+    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="7.0.9" />
-      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
-      <PrivateAssets>all</PrivateAssets>
-    </PackageReference>
+    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="7.0.9" />
-      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
-      <PrivateAssets>all</PrivateAssets>
-    </PackageReference>
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="7.0.4" />
  </ItemGroup>
```

## Verify the library class is built successfully
- via visual studio code, run below command to ensure project is built successfully

``` bash
dotnet build
```