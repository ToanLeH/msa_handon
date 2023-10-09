# Sync Data from Product Service to Order Service through Domain Event

## Create Events folder via MSA.Common.Contracts project
- via visual studio code, create **Events** folder via MSA.Common.Contracts project

### Create Product folder via **Events** folder 
- via visual studio code, create **Product** folder via **Events** folder 

#### Create **ProductEvents.cs** file via **Product** folder
- Create **ProductEvents.cs** file via **Product** folder and paste below code to it
``` C#
namespace MSA.Common.Contracts.Domain.Events.Product;

public record ProductCreated
{
    public Guid ProductId { get; init; }
};
```

## Modify MSA.ProductService to publish ProductCreated Event
### Add reference to MSA.Common.PostgresMassTransit to integrate with RabbitMQ
- via Visual studio code, run below command to add reference to MSA.Common.PostgresMassTransit
``` bash
dotnet add reference ../MSA.Common.PostgresMassTransit
```

### Modify Program.cs file to integrate with MassTransit RabbitMQ
- Modify Program.cs as below
``` Diff
using MSA.ProductService.Entities;
using MSA.Common.MongoDB;
using MSA.Common.Contracts.Domain;
+ using MSA.Common.PostgresMassTransit.MassTransit;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddMongo()
                .AddRepositories<Product>("product")
+                 .AddMassTransitWithRabbitMQ();

builder.Services.AddControllers(options => {
    options.SuppressAsyncSuffixInActionNames = false;
});
```

### Modify AppSettings.json to include RabbitMQ configuration
- Modify AppSetting.json as below
``` Diff
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ServiceSetting": {
      "ServiceName": "Product"
    },
    "MongoDBSetting": {
      "Host": "localhost",
      "Port": "27017"
    },
+  "AllowedHosts": "*",
+  "RabbitMQSetting": {
+    "Host": "localhost"
+  }
}

```

### Modify ProductController.cs to publish ProductCreated Domain Event in case users create new product
- modify ProductController.cs based on below
``` Diff
using Microsoft.AspNetCore.Mvc;
using MSA.ProductService.Dtos;
using MSA.ProductService.Entities;
using MSA.Common.Contracts.Domain;
+ using MassTransit;
+ using MSA.Common.Contracts.Domain.Events.Product;

namespace MSA.ProductService.Controllers
{
    [ApiController]
    [Route("v1/product")]
    public class ProductController : ControllerBase
    {
        private readonly IRepository<Product> _repository;
+         private readonly IPublishEndpoint publishEndpoint;

        public ProductController(
+             IRepository<Product> repository,
+             IPublishEndpoint publishEndpoint)
        {
            this._repository = repository;
+             this.publishEndpoint = publishEndpoint;
        }

        [HttpGet]
        public async Task<IEnumerable<ProductDto>> GetAsync()
        {
            var products = (await _repository.GetAllAsync())
                            .Select(p => p.AsDto());
            return products;
        }

        //Get v1/product/123
        [HttpGet("{id}")]
        public async Task<ActionResult<Guid>> GetByIdAsync(Guid id)
        {
            if (id == null) return BadRequest();

            var product = (await _repository.GetAsync(id));
            if (product == null) return Ok(Guid.Empty);

            return Ok(product.Id);
        }

        [HttpPost]
        public async Task<ActionResult<ProductDto>> PostAsync(
            CreateProductDto createProductDto)
        {
            var product = new Product
            {
                Id = new Guid(),
                Name = createProductDto.Name,
                Description = createProductDto.Description,
                Price = createProductDto.Price,
                CreatedDate = DateTimeOffset.UtcNow
            };
            await _repository.CreateAsync(product);

+             await publishEndpoint.Publish(new ProductCreated{
+                 ProductId = product.Id
+             });

            return CreatedAtAction(nameof(PostAsync), product.AsDto());
        }
    }
}
```

### Build projcet
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```

## Modify MSA.OrderService to consume ProductCreated Event 
### Modify Program.cs file to integrate with MassTransit RabbitMQ 
``` Diff
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.Common.Contracts.Settings;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Services;
+ using MSA.Common.PostgresMassTransit.MassTransit;

var builder = WebApplication.CreateBuilder(args);

PostgresDBSetting serviceSetting = builder.Configuration.GetSection(nameof(PostgresDBSetting)).Get<PostgresDBSetting>();


// Add services to the container.
builder.Services
    .AddPostgres<MainDbContext>()
    .AddPostgresRepositories<MainDbContext, Order>()
    .AddPostgresUnitofWork<MainDbContext>()
+     .AddMassTransitWithRabbitMQ();
```
### Modify AppSetting.json to include RabbitMQ configuration
``` Diff
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "PostgresDBSetting": {
    "ConnectionString": "User ID=guest;Password=guest;Server=localhost;Port=5432;Database=msapostgres"
  },
  "ServiceSetting": {
      "ServiceName": "Order"
+  },
+  "RabbitMQSetting": {
+    "Host": "localhost"
+  }
}
```
### Create new Entity Product.cs via Domain folder and add migration to it
- Create new Entity **Product.cs** file via Domain folder and paste below code to it
``` C#
using MSA.Common.Contracts.Domain;

namespace MSA.OrderService.Domain;

public class Product : IEntity
{
    public Guid Id { get; set; }
    public Guid ProductId { get; set; }
}
```
- Configuration for Entity via Infrastructure/Data/MainDbContext.cs
``` Diff
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

+            //Product
+            modelBuilder.Entity<Product>().ToTable("products");
+            modelBuilder.Entity<Product>().HasKey(x => x.Id);
+            modelBuilder.Entity<Product>().Property(x => x.Id)
+                .HasColumnType("uuid");
            
            //Relationship
            modelBuilder.Entity<Order>()
                .HasMany(x => x.OrderDetails);
        }
```

- Add Migrations for new Entity via below command
``` bash
dotnet ef migrations add add_product_table -c MainDbContext
```

- Update database based on new migration
``` bash
dotnet ef database update -c MainDbContext
```

- Check Postgres Extension via visual studio, ensure **products** table exist in **order** table

### Register Repository for Product.cs Entity
- Register Repository for Product.cs Entity
``` Diff
// Add services to the container.
builder.Services
    .AddPostgres<MainDbContext>()
    .AddPostgresRepositories<MainDbContext, Order>()
+    .AddPostgresRepositories<MainDbContext, Product>()
    .AddPostgresUnitofWork<MainDbContext>()
    .AddMassTransitWithRabbitMQ();
```

### Create ProductCreatedConsumer.cs to consume ProductCreated Event
- Create **Consumers** folder via MSA.OrderService Project
- Create **ProductCreatedConsumer.cs** file via **Consumers** folder and paste below code to it
``` C#
using MassTransit;
using MSA.Common.Contracts.Domain;
using MSA.Common.Contracts.Domain.Events.Product;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;

namespace MSA.OrderService.Consumers;

public class ProductCreatedConsumer : IConsumer<ProductCreated>
{
    private readonly IRepository<Product> productRepository;
    private readonly PostgresUnitOfWork<MainDbContext> uoW;

    public ProductCreatedConsumer(
        IRepository<Product> productRepository,
        PostgresUnitOfWork<MainDbContext> uoW)
    {
        this.productRepository = productRepository;
        this.uoW = uoW;
    }

    public async Task Consume(ConsumeContext<ProductCreated> context)
    {
        var message = context.Message;
        Product product = new Product {
            Id = new Guid(),
            ProductId = message.ProductId
        };
        await productRepository.CreateAsync(product);
        await uoW.SaveChangeAsync();
    }
}
```

### Build projcet
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```

## Verify flow
- start Order Service and Product Service, run below command
``` bash
dotnet run
```

### Verify Exchange and Queue are created successfully
- Log in to RabbitMQ Admin Portal **http://localhost:15672/#** via username password **guest/guest**
- Ensure **MSA.Common.Contracts.Domain.Events.Product:ProductCreated** Exchange is created
- Ensure **order-product-created** queue is created

### Try to submit one product and ensure data is sync to order postgres database
- verify via swagger to create new product
- Check Postgres Database and ensure data is sync here

## [TODO-HOMEWORK] Modify OrderController.cs to use Repository to check product exist or not