# Handle token-based authentication and policy-based authorization for product Service swagger

## Add reference to MSA.Common.Security project
- via Visual studio code, run below command to add reference to MSA.Common.Security
``` bash
dotnet add reference ../MSA.Common.Security
```

## Modify **Program.cs** to add authentication using Duende Identity Server
- Modify **Program.cs** with below code
``` Diff
using MSA.ProductService.Entities;
using MSA.Common.MongoDB;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.MassTransit;
+ using MSA.Common.Security.Authentication;
+ using MSA.Common.Security.Authorization;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddMongo()
                .AddRepositories<Product>("product")
                .AddMassTransitWithRabbitMQ()
+                 .AddMSAAuthentication()
+                 .AddMSAAuthorization(opt => {
+                     opt.AddPolicy("read_access", policy => {
+                         policy.RequireClaim("scope", "productapi.read");
+                     });
+                 });

builder.Services.AddControllers(options => {
    options.SuppressAsyncSuffixInActionNames = false;
});

builder.Services.AddControllers();
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

//app.UseHttpsRedirection();

+ app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();

```

## Modify **Program.cs** to add jwt authentication for swagger 
- Modify **Program.cs** to add jwt authentication for swagger 
``` Diff
using MSA.ProductService.Entities;
using MSA.Common.MongoDB;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.MassTransit;
using MSA.Common.Security.Authentication;
using MSA.Common.Security.Authorization;
+ using Microsoft.AspNetCore.Authentication.JwtBearer;
+ using Microsoft.OpenApi.Models;
+ using MSA.Common.Contracts.Settings;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddMongo()
                .AddRepositories<Product>("product")
                .AddMassTransitWithRabbitMQ()
                .AddMSAAuthentication()
                .AddMSAAuthorization(opt => {
                    opt.AddPolicy("read_access", policy => {
                        policy.RequireClaim("scope", "productapi.read");
                    });
                });

builder.Services.AddControllers(options => {
    options.SuppressAsyncSuffixInActionNames = false;
});

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
+ //builder.Services.AddSwaggerGen();

+ var srvUrlsSetting = builder.Configuration.GetSection(nameof(ServiceUrlsSetting)).Get<ServiceUrlsSetting>();
+ builder.Services.AddSwaggerGen(options =>
+ {
+     var scheme = new OpenApiSecurityScheme
+     {
+         In = ParameterLocation.Header,
+         Name = "Authorization",
+         Flows = new OpenApiOAuthFlows
+         {
+             AuthorizationCode = new OpenApiOAuthFlow
+             {
+                 AuthorizationUrl = new Uri($"{srvUrlsSetting.IdentityServiceUrl}/connect/authorize"),
+                 TokenUrl = new Uri($"{srvUrlsSetting.IdentityServiceUrl}/connect/token"),
+                 Scopes = new Dictionary<string, string>
+                 {
+                     { "productapi.read", "Access read operations" },
+                     { "productapi.write", "Access write operations" }
+                 }
+             }
+         },
+         Type = SecuritySchemeType.OAuth2
+     };
+ 
+     options.AddSecurityDefinition("OAuth", scheme);
+ 
+     options.AddSecurityRequirement(new OpenApiSecurityRequirement
+     {
+         { 
+             new OpenApiSecurityScheme
+             {
+                 Reference = new OpenApiReference { Id = "OAuth", Type = ReferenceType.SecurityScheme }
+             }, 
+             new List<string> { } 
+         }
+     });
+ });

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
+   //app.UseSwaggerUI();
+     app.UseSwaggerUI(options =>
+     {
+         options.OAuthClientId("product-swagger");
+         options.OAuthScopes("profile", "openid");
+     });
}

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();

```

## Modify **appsetting.json** to include service url setting
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
  "AllowedHosts": "*",
  "RabbitMQSetting": {
    "Host": "localhost"
+  },
+   "ServiceUrlsSetting": {
+     "IdentityServiceUrl": "https://localhost:5001",
+     "ProductServiceUrl": "",
+     "OrderServiceUrl": ""
+   }
}

```

## Modify **ProductController** to authorize endpoint
``` Diff
using Microsoft.AspNetCore.Mvc;
using MSA.ProductService.Dtos;
using MSA.ProductService.Entities;
using MSA.Common.Contracts.Domain;
using MassTransit;
using MSA.Common.Contracts.Domain.Events.Product;
+ using Microsoft.AspNetCore.Authorization;

namespace MSA.ProductService.Controllers
{
    [ApiController]
    [Route("v1/product")]
+     [Authorize]
    public class ProductController : ControllerBase
    {
        private readonly IRepository<Product> _repository;
        private readonly IPublishEndpoint publishEndpoint;

        public ProductController(
            IRepository<Product> repository,
            IPublishEndpoint publishEndpoint)
        {
            this._repository = repository;
            this.publishEndpoint = publishEndpoint;
        }

        [HttpGet]
+         [Authorize("read_access")]
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

            await publishEndpoint.Publish(new ProductCreated{
                ProductId = product.Id
            });

            return CreatedAtAction(nameof(PostAsync), product.AsDto());
        }
    }
}
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
- Browse to url https://localhost:5002/swagger and try to call Get endpoint with and without authentication