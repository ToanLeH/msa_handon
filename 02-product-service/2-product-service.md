# Create an MSA.ProductService project skeleton that use Mongo Database

# Create MSA.ProductService web api project
## Create new webapi project
- via Visual studio code, run below command to create new webapi project
``` bash
dotnet new webapi -n MSA.ProductService
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.ProductService
```

## Add reference to MSA.Common.Contracts project
- via Visual studio code, run below command to add reference to MSA.Common.Contracts and MSA.Common.MongoDB
``` bash
dotnet add reference ../MSA.Common.Contracts
dotnet add reference ../MSA.Common.Mongo
```

## Create Entities folder via MSA.ProductService web api project
- via visual studio code, create **Entities** folder via MSA.ProductService web api project

### Create **Product.cs** file via Entities Folder
- via visual studio code, create **Product.cs** folder via **Entities** folder and paste below code to it
``` C#
using System;
using MSA.Common.Contracts.Domain;

namespace MSA.ProductService.Entities
{
    public class Product : IEntity
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public string Description { get; set; }
        public decimal Price { get; set; }
        public DateTimeOffset CreatedDate { get; set; }
    }
}
```

## Create Dtos folder via MSA.ProductService web api project
- via visual studio code, create **Dtos** folder via MSA.ProductService web api project

### Create **ProductDto.cs** file via Dtos Folder
- via visual studio code, create **ProductDto.cs** folder via **Dtos** folder and paste below code to it
``` C#
using System.ComponentModel.DataAnnotations;

namespace MSA.ProductService.Dtos
{
    public record ProductDto
    (
        Guid Id,
        string Name,
        string Description,
        decimal Price,
        DateTimeOffset CreatedDate
    );

    public record CreateProductDto
    (
        [Required] string Name,
        [Required] string Description,
        [Range(0,1000)] Decimal Price
    );
}
```

### Create **Extensions.cs** file via Dtos Folder
- via visual studio code, create **Extensions.cs** folder via **Dtos** folder and paste below code to it
``` C#
using MSA.ProductService.Entities;

namespace MSA.ProductService.Dtos
{
    public static class Extensions
    {
        public static ProductDto AsDto(this Product product)
        {
            return new ProductDto(
                product.Id,
                product.Name,
                product.Description,
                product.Price,
                product.CreatedDate
            );
        }
    }
}
```

## Create **ProductController.cs** via Controllers folder
- via visual studio code, create **ProductController.cs** folder via **Controllers** folder
``` C#
using Microsoft.AspNetCore.Mvc;
using MSA.ProductService.Dtos;
using MSA.ProductService.Entities;
using MSA.Common.Contracts.Domain;

namespace MSA.ProductService.Controllers
{
    [ApiController]
    [Route("v1/product")]
    public class ProductController : ControllerBase
    {
        private readonly IRepository<Product> _repository;

        public ProductController(
            IRepository<Product> repository)
        {
            this._repository = repository;
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

            return CreatedAtAction(nameof(PostAsync), product.AsDto());
        }
    }
}
```

## Modify **Program.cs** file to integrate with MongoDB
- Modify **Program.cs** file to integrate with MongoDB
``` Diff
+ using MSA.ProductService.Entities;
+ using MSA.Common.MongoDB;
+ using MSA.Common.Contracts.Domain;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
+ builder.Services.AddMongo()
+                 .AddRepositories<Product>("product");

+ builder.Services.AddControllers(options => {
+     options.SuppressAsyncSuffixInActionNames = false;
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

## Modify **LaunchSettings.json** to run the project via port 5002
``` Diff
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5002;http://localhost:15002",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5002;http://localhost:15002",
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
+  "ServiceSetting": {
+    "ServiceName": "Product"
+  },
+  "MongoDBSetting": {
+    "Host": "localhost",
+    "Port": "27017"
+  },
  "AllowedHosts": "*"
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
- Browse to url https://localhost:5002/swagger