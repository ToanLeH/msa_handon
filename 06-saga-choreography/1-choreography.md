# Async Communication between Order Service and Product Service using Saga Choreography Pattern

## Create Commands folder via MSA.Common.Contracts project
- via visual studio code, create **Commands** folder via MSA.Common.Contracts project

### Create Product folder via **Commands** folder 
- via visual studio code, create **Product** folder via **Commands** folder 

#### Create **ProductCommands.cs** file via **Commands/Product** folder
- Create **ProductCommands.cs** file via **Commands/Product** folder and paste below code to it
``` C#
namespace MSA.Common.Contracts.Domain.Commands.Product;

public record ValidateProduct
{
    public Guid OrderId { get; init; }
    public Guid ProductId { get; init; }
}
```

## Modify MSA.OrderService to Send validateProduct Command
### Modify OrderController.cs to send ValidateProduct Command 
``` Diff
using Microsoft.AspNetCore.Mvc;
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.OrderService.Dtos;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Services;
+ using MassTransit;
+ using MSA.Common.Contracts.Domain.Commands.Product;

namespace MSA.OrderService.Controllers;

[ApiController]
[Route("v1/order")]
public class OrderController : ControllerBase
{
    private readonly IRepository<Order> repository;

    private readonly PostgresUnitOfWork<MainDbContext> uow;
    private readonly IProductService productService;
+    private readonly ISendEndpointProvider sendEndpointProvider;

    public OrderController(
        IRepository<Order> repository,
        PostgresUnitOfWork<MainDbContext> uow,
        IProductService productService,
+        ISendEndpointProvider sendEndpointProvider
        )
    {
        this.repository = repository;
        this.uow = uow;
        this.productService = productService;
+        this.sendEndpointProvider = sendEndpointProvider;
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
+        //validate and ensure product exist before creating
+        //var isProductExisted = await productService.IsProductExisted(createOrderDto.ProductId);
+        //if (!isProductExisted) return BadRequest();

        var order = new Order { 
            Id = Guid.NewGuid(),
            UserId = createOrderDto.UserId,
            OrderStatus = "Order Submitted"
        };
        await repository.CreateAsync(order);

        await uow.SaveChangeAsync();

+        //async validate
+        var endpoint = await sendEndpointProvider.GetSendEndpoint(
+            new Uri("queue:product-validate-product")
+        );
+        await endpoint.Send(new ValidateProduct{
+            OrderId = order.Id,
+            ProductId = createOrderDto.ProductId
+        });

        return CreatedAtAction(nameof(PostAsync), order);
    }
}
```

### Build projcet
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```

## Modify MSA.ProductService to consume validateProduct Command
- Create **Consumers** folder via MSA.ProductService Project
- Create **ValidateProductConsumer.cs** file via **Consumers** folder and paste below code to it
``` C#
using MassTransit;
using MSA.Common.Contracts.Domain;
using MSA.Common.Contracts.Domain.Commands.Product;
using MSA.ProductService.Entities;

namespace MSA.ProductService.Consumers;

public class ValidateProductConsumer : IConsumer<ValidateProduct>
{
    private readonly ILogger<ValidateProductConsumer> logger;
    private readonly IRepository<Product> repository;

    public ValidateProductConsumer(
        ILogger<ValidateProductConsumer> logger,
        IRepository<Product> repository)
    {
        this.logger = logger;
        this.repository = repository;
    }

    public async Task Consume(ConsumeContext<ValidateProduct> context)
    {
        var message = context.Message;
        //TODO : Validate and submit Commands/Events for further flow
        logger.Log(LogLevel.Information, 
            $"Receiving message of order {message.OrderId} validating product {message.ProductId}"
        );
    }
}
```

### Build projcet
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```

### Verify 
- Using Order Service swagger to create new product and ensure you see command is proceed by Product Service