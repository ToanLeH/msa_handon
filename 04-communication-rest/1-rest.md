# Implement REST communication style between Order Service and Product Service

# Modify **MSA.OrderService** service
## Create **Services** Folder
- via visual studio code, create **Services** folder

### Create **IProductService.cs** file via Services Folder
- via visual studio code, create **IProductService.cs** file via **Services** folder and paste below code to it
``` C#
namespace MSA.OrderService.Services;

public interface IProductService
{
    Task<bool> IsProductExisted(Guid id);
}

```

### Create **ProductService.cs** file via Services Folder
- via visual studio code, create **ProductService.cs** file via **Services** folder and paste below code to it
``` C#
using System.Diagnostics.Contracts;
namespace MSA.OrderService.Services;

public class ProductService : IProductService
{
    private readonly HttpClient httpClient;

    public ProductService(HttpClient httpClient)
    {
        this.httpClient = httpClient;
    }

    public async Task<bool> IsProductExisted(Guid id)
    {
        var result = await httpClient.GetStringAsync($"v1/product/{id}");
        var existedId = Guid.Empty;
        Guid.TryParse(result, out existedId);
        if (existedId == id) return await Task.FromResult(true);

        return await Task.FromResult(false);
    }
}
```

### Modify **program.cs** to register HttpClient
- Modify **Program.cs** file as below
``` Diff
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.Common.Contracts.Settings;
using MSA.Common.PostgresMassTransit.PostgresDB;
+ using MSA.OrderService.Services;

var builder = WebApplication.CreateBuilder(args);

PostgresDBSetting serviceSetting = builder.Configuration.GetSection(nameof(PostgresDBSetting)).Get<PostgresDBSetting>();


// Add services to the container.
builder.Services
    .AddPostgres<MainDbContext>()
    .AddPostgresRepositories<MainDbContext, Order>()
    .AddPostgresUnitofWork<MainDbContext>();

+ builder.Services.AddHttpClient<IProductService, ProductService>(cfg => {
+     cfg.BaseAddress = new Uri("https://localhost:5002");
+ });

builder.Services.AddControllers(opt => {
    opt.SuppressAsyncSuffixInActionNames = false;
});
```

### Modify **OrderController.cs** to inject ProductService to it
- Modify **OrderController.cs** file as below
``` Diff
using Microsoft.AspNetCore.Mvc;
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.OrderService.Dtos;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.PostgresDB;
+ using MSA.OrderService.Services;

namespace MSA.OrderService.Controllers;

[ApiController]
[Route("v1/order")]
public class OrderController : ControllerBase
{
    private readonly IRepository<Order> repository;

    private readonly PostgresUnitOfWork<MainDbContext> uow;
+     private readonly IProductService productService;

    public OrderController(
        IRepository<Order> repository,
+         PostgresUnitOfWork<MainDbContext> uow,
+         IProductService productService
        )
    {
        this.repository = repository;
        this.uow = uow;
+         this.productService = productService;
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
+         //validate and ensure product exist before creating
+         var isProductExisted = await productService.IsProductExisted(createOrderDto.ProductId);
+         if (!isProductExisted) return BadRequest();

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
- Browse to url http://localhost:5003/swagger
- Try to submit an Order with non-existing product, ensure you receive 400 Bad Request Status Code