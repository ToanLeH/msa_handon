# Create an MSA.OrderService project skeleton that use Postgres Database

# Create MSA.OrderService web api project
## Create new webapi project
- via Visual studio code, run below command to create new webapi project
``` bash
dotnet new webapi -n MSA.OrderService
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.OrderService
```

## Add reference to MSA.Common.Contracts project
- via Visual studio code, run below command to add reference to MSA.Common.Contracts and MSA.Common.PostgresMassTranssit
``` bash
dotnet add reference ../MSA.Common.Contracts
dotnet add reference ../MSA.Common.PostgresMassTransit
```

## Create Domain folder via MSA.OrderService web api project
- via visual studio code, create **Domain** folder via MSA.OrderService web api project

### Create **Order.cs** file via Domain Folder
- via visual studio code, create **Order.cs** folder via **Domain** folder and paste below code to it
``` C#
using MSA.Common.Contracts.Domain;

namespace MSA.OrderService.Domain
{
    public class Order : IEntity
    {
        public Guid Id { get; set; }
        public Guid UserId { get; set; }
        public string OrderStatus { get; set; }
        public virtual ICollection<OrderDetail> OrderDetails { get; set; }
    }
}
```

### Create **OrderDetail.cs** file via Domain Folder
- via visual studio code, create **OrderDetail.cs** folder via **Domain** folder and paste below code to it
``` C#
using MSA.Common.Contracts.Domain;

namespace MSA.OrderService.Domain
{
    public class OrderDetail : IEntity
    {
        public Guid Id { get; set; }
        public Guid ProductId { get; set; }
    }
}
```

## Create Infrastructure folder via MSA.OrderService web api project
- via visual studio code, create **Infrastructure** folder via MSA.OrderService web api project

### Create Data folder via Infrastructure folder
- via visual studio code, create **Data** folder via **Infrastructure** folder

#### Create **MainDbContext.cs** file via Data Folder
- via visual studio code, create **MainDbContext.cs** folder via **Data** folder and paste below code to it
``` C#
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Domain;
using Microsoft.EntityFrameworkCore;

namespace MSA.OrderService.Infrastructure.Data
{
    public class MainDbContext : AppDbContextBase
    {
        private readonly string _uuidGenerator = "uuid-ossp";
        private readonly string _uuidAlgorithm = "uuid_generate_v4()";
        private readonly IConfiguration configuration;

        public MainDbContext(
            IConfiguration configuration,
            DbContextOptions<MainDbContext> options) : base(configuration, options)
        {
            this.configuration = configuration;
        }

        public DbSet<Order> Orders { get; set; } = default!;
        public DbSet<OrderDetail> OrderDetails { get; set; } = default!;

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            modelBuilder.HasPostgresExtension(_uuidGenerator);

            //Orders
            modelBuilder.Entity<Order>().ToTable("orders");
            modelBuilder.Entity<Order>().HasKey(x => x.Id);
            modelBuilder.Entity<Order>().Property(x => x.Id)
                .HasColumnType("uuid");
                //.HasDefaultValueSql(_uuidAlgorithm);

            //Order Details
            modelBuilder.Entity<OrderDetail>().ToTable("order_details");
            modelBuilder.Entity<OrderDetail>().HasKey(x => x.Id);
            modelBuilder.Entity<OrderDetail>().Property(x => x.Id)
                .HasColumnType("uuid")
                .HasDefaultValueSql(_uuidAlgorithm);

            //Relationship
            modelBuilder.Entity<Order>()
                .HasMany(x => x.OrderDetails);
        }
    }
}
```

## Create Dtos folder via MSA.OrderService web api project
- via visual studio code, create **Dtos** folder via MSA.OrderService web api project

### Create **OrderDto.cs** file via Dtos Folder
- via visual studio code, create **OrderDto.cs** folder via **Dtos** folder and paste below code to it
``` C#
namespace MSA.OrderService.Dtos;

public record CreateOrderDto
(
    Guid UserId,
    Guid ProductId
);
```

## Create **OrderController.cs** via Controllers folder
- via visual studio code, create **OrderController.cs** folder via **Controllers** folder
``` C#
using Microsoft.AspNetCore.Mvc;
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.OrderService.Dtos;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.PostgresDB;

namespace MSA.OrderService.Controllers;

[ApiController]
[Route("v1/order")]
public class OrderController : ControllerBase
{
    private readonly IRepository<Order> repository;

    private readonly PostgresUnitOfWork<MainDbContext> uow;

    public OrderController(
        IRepository<Order> repository,
        PostgresUnitOfWork<MainDbContext> uow
        )
    {
        this.repository = repository;
        this.uow = uow;
    }

    [HttpGet]
    public async Task<IEnumerable<Order>> GetAsync()
    {
        var orders = (await repository.GetAllAsync()).ToList();
        return orders;
    }

    [HttpPost]
    public async Task<ActionResult<Order>> PostAsync(CreateOrderDto createOrderDto)
    {
        var order = new Order { 
            Id = Guid.NewGuid(),
            UserId = createOrderDto.UserId,
            OrderStatus = "Order Submitted"
        };
        await repository.CreateAsync(order);

        await uow.SaveChangeAsync();

        return CreatedAtAction(nameof(PostAsync), order);
    }
}
```

## Modify **Program.cs** file to integrate with Postgres
- Modify **Program.cs** file to integrate with Postgres

``` Diff
+ using MSA.OrderService.Domain;
+ using MSA.OrderService.Infrastructure.Data;
+ using MSA.Common.Contracts.Settings;
+ using MSA.Common.PostgresMassTransit.PostgresDB;

var builder = WebApplication.CreateBuilder(args);

+ PostgresDBSetting serviceSetting = builder.Configuration.GetSection(nameof(PostgresDBSetting)).Get<PostgresDBSetting>();


// Add services to the container.
+ builder.Services
+     .AddPostgres<MainDbContext>()
+     .AddPostgresRepositories<MainDbContext, Order>()
+     .AddPostgresUnitofWork<MainDbContext>();

+ builder.Services.AddControllers(opt => {
+     opt.SuppressAsyncSuffixInActionNames = false;
+ });
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

## Modify **LaunchSettings.json** to run the project via port 5003
``` Diff
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5003;http://localhost:15003",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5003;http://localhost:15003",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
```

## Add below configuration to appsettings.json file
``` Diff
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
+  "AllowedHosts": "*",
+  "PostgresDBSetting": {
+    "ConnectionString": "User ID=guest;Password=guest;Server=localhost;Port=5432;Database=msapostgres"
+  },
+  "ServiceSetting": {
+    "ServiceName": "Order"
+  }
}
```

## Add Instruct Kestrel to use local self-signed certificate for appsettings.json
``` Diff
+  "Kestrel": {
+    "Certificates": {
+       "Default": {
+         "Path": "../aspnet/https/localhost.pfx",
+         "Password": "MsaFundamental"
+       }
+     }
+   }
```

## Add Migrations to project
- via visual studio code, run below command to add migrations for entity framework
``` bash
dotnet ef migrations add init_database -c MainDbContext
dotnet ef database update
```

## Build and Run Project
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```
- Run below command to run project 
``` bash
dotnet run
```

## Browse to below url and ensure you get the response on it
- Browse to url https://localhost:5003/swagger