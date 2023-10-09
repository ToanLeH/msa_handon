# coordinate all services communication with synchronize status via Saga orchestration Pattern

## Create **OrderSubmitted** event 
- Create folder **Order** via **MSA.Common.Contracts\Events** folder
- Create **OrderEvent.cs** file via **MSA.Common.Contracts\Events\Order* folder and paste below code to it
``` C#
namespace MSA.Common.Contracts.Domain.Events.Order;

public record OrderSubmitted
{
    public Guid OrderId { get; init; }
    public Guid ProductId { get; init; }
};
```

## Modify **ProductEvents** 
- Modify file **ProductEvents.cs** via **MSA.Common.Contracts\Domain\Events\Product** and paste below code to it
``` Diff
namespace MSA.Common.Contracts.Domain.Events.Product;

public record ProductCreated
{
    public Guid ProductId { get; init; }
};

+ public record ProductValidatedSucceeded 
+ {
+     public Guid OrderId { get; init; }
+     public Guid ProductId { get; init; }
+ } 
+ 
+ public record ProductValidatedFailed
+ {
+     public Guid OrderId { get; init; }
+     public Guid ProductId { get; init; }
+     public string Reason { get; init; }
+ } 
```

## Create **OrderStateMachine** to orchestrate the order flow
- Create folder **StateMachine** via project **MSA.OrderService**
- Create file **OrderState.cs** via **StateMachine** folder and paste below content to it

``` C#
using MassTransit;

namespace MSA.OrderService.StateMachine;

public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public Guid? OrderId { get; set; }

    public Guid? PaymentId { get; set; }
    public string? Reason { get; set; }

    public Guid? ProductValidationId { get; set; }

    public string CurrentState { get; set; }
}
```

- Create file **OrderStateMachine.cs** via **StateMachine** folder and paste below content to it

``` C#
using MassTransit;
using MSA.Common.Contracts.Settings;
using MSA.Common.Contracts.Domain.Events.Order;
using MSA.Common.Contracts.Domain.Events.Product;

namespace MSA.OrderService.StateMachine;

public class OrderStateMachine 
    : MassTransitStateMachine<OrderState>
{
    private readonly IConfiguration configuration;

    public OrderStateMachine(
        IConfiguration configuration
    )
    {
        this.configuration = configuration;

        var serviceEndPoints = configuration
            .GetSection(nameof(ServiceEndpointSettings))
            .Get<ServiceEndpointSettings>();

        InstanceState(
            x => x.CurrentState
        );

        Event(() => OrderSubmitted,
            x => x.CorrelateById(context => context.Message.OrderId)
        );

        Event(() => ProductValidatedSucceeded,
            x => x.CorrelateById(context => context.Message.OrderId)
        );

        Event(() => ProductValidatedFailed,
            x => x.CorrelateById(context => context.Message.OrderId)
        );

        Initially(
            When(OrderSubmitted)
                .Then(x => Console.WriteLine($"Receiving Order {x.Message.OrderId}"))
                .Then(x => {
                    x.Saga.OrderId = x.Message.OrderId;
                })
                .TransitionTo(Submitted)
        );

        During(Submitted,
            When(ProductValidatedSucceeded)
                .Then(x => Console.WriteLine($"Validate Product Succeeded for OrderId {x.Message.OrderId}"))
                .Then(x => {
                    x.Saga.ProductValidationId = x.Message.ProductId;
                })
                .TransitionTo(ValidatedSucceeded)
                .Finalize(),
            When(ProductValidatedFailed)
                .Then(x => Console.WriteLine($"Validate Product Failed for OrderId {x.Message.OrderId}"))
                .Then(x => {
                    x.Saga.ProductValidationId = x.Message.ProductId;
                    x.Saga.Reason = x.Message.Reason;
                })
                .TransitionTo(ValidatedSucceeded)
                .Finalize()
        );
 
    }

    public Event<OrderSubmitted> OrderSubmitted { get; private set; }
    public Event<ProductValidatedSucceeded> ProductValidatedSucceeded { get; private set; }
    public Event<ProductValidatedFailed> ProductValidatedFailed { get; private set; }

    public State Submitted { get; private set; }
    public State ValidatedSucceeded { get; private set; }
    public State ValidatedFailed { get; private set; }
}
```

## Create **OrderStateDbContext** to persist the whole order flow
- Create folder **Saga** via folder **Infrastructure**
- Create file **OrderStateMap.cs** and paste below code to it
``` C#
using MassTransit;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using MSA.OrderService.StateMachine;

namespace MSA.OrderService.Infrastructure.Saga;

public class OrderStateMap : 
    SagaClassMap<OrderState>
{
    protected override void Configure(EntityTypeBuilder<OrderState> entity, ModelBuilder model)
    {
        entity.Property(x => x.CurrentState).HasMaxLength(64);
        entity.Property(x => x.OrderId);
        entity.Property(x => x.PaymentId);
        entity.Property(x => x.Reason);
    }
}
```

- Create file **OrderStateDbContext.cs** and paste below code to it
``` C#
using MassTransit.EntityFrameworkCoreIntegration;
using Microsoft.EntityFrameworkCore;

namespace MSA.OrderService.Infrastructure.Saga;

public class OrderStateDbContext : 
    SagaDbContext
{
    public OrderStateDbContext(DbContextOptions<OrderStateDbContext> options)
        : base(options)
    {
    }

    protected override IEnumerable<ISagaClassMap> Configurations
    {
        get { yield return new OrderStateMap(); }
    }
}
```

## Modify **Program.cs** to integrate and register OrderStateMachine
- Modify **Program.cs** as below
``` Diff
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.Common.Contracts.Settings;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Services;
using MSA.Common.PostgresMassTransit.MassTransit;
+ using MSA.OrderService.StateMachine;
+ using MassTransit;
+ using MassTransit.EntityFrameworkCoreIntegration;
+ using Microsoft.EntityFrameworkCore;
+ using MSA.OrderService.Infrastructure.Saga;
+ using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

PostgresDBSetting serviceSetting = builder.Configuration.GetSection(nameof(PostgresDBSetting)).Get<PostgresDBSetting>();


// Add services to the container.
builder.Services
    .AddPostgres<MainDbContext>()
    .AddPostgresRepositories<MainDbContext, Order>()
    .AddPostgresRepositories<MainDbContext, Product>()
    .AddPostgresUnitofWork<MainDbContext>()
+     //.AddMassTransitWithRabbitMQ();
+     .AddMassTransitWithPostgresOutbox<MainDbContext>(cfg => {
+         cfg.AddSagaStateMachine<OrderStateMachine, OrderState>()
+            .EntityFrameworkRepository(r => {
+                 r.ConcurrencyMode = ConcurrencyMode.Pessimistic;
+ 
+                 r.LockStatementProvider = new PostgresLockStatementProvider();
+ 
+                 r.AddDbContext<DbContext, OrderStateDbContext>((provider,builder) =>
+                 {
+                     builder.UseNpgsql(serviceSetting.ConnectionString,n => {
+                         n.MigrationsAssembly(Assembly.GetExecutingAssembly().GetName().Name);
+                         n.MigrationsHistoryTable($"__{nameof(OrderStateDbContext)}");
+                     });
+                 });
+            });
+     });
```

- Run below command to add migrations for new context
``` bash
dotnet ef migrations add InitialCreate -c OrderStateDbContext
dotnet ef database update -c OrderStateDbContext
```

## Modify **OrderController.cs** to trigger the Saga Workflow
- Modify **OrderController.cs** to trigger the Saga Workflow
``` Diff
using Microsoft.AspNetCore.Mvc;
using MSA.OrderService.Domain;
using MSA.OrderService.Infrastructure.Data;
using MSA.OrderService.Dtos;
using MSA.Common.Contracts.Domain;
using MSA.Common.PostgresMassTransit.PostgresDB;
using MSA.OrderService.Services;
using MassTransit;
using MSA.Common.Contracts.Domain.Commands.Product;
+ using MSA.Common.Contracts.Domain.Events.Order;

namespace MSA.OrderService.Controllers;

[ApiController]
[Route("v1/order")]
public class OrderController : ControllerBase
{
    private readonly IRepository<Order> repository;

    private readonly PostgresUnitOfWork<MainDbContext> uow;
    private readonly IProductService productService;
    private readonly ISendEndpointProvider sendEndpointProvider;
+     private readonly IPublishEndpoint publishEndpoint;

    public OrderController(
        IRepository<Order> repository,
        PostgresUnitOfWork<MainDbContext> uow,
        IProductService productService,
        ISendEndpointProvider sendEndpointProvider,
+         IPublishEndpoint publishEndpoint
        )
    {
        this.repository = repository;
        this.uow = uow;
        this.productService = productService;
        this.sendEndpointProvider = sendEndpointProvider;
+         this.publishEndpoint = publishEndpoint;
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
        //validate and ensure product exist before creating
        //var isProductExisted = await productService.IsProductExisted(createOrderDto.ProductId);
        //if (!isProductExisted) return BadRequest();

        var order = new Order { 
            Id = Guid.NewGuid(),
            UserId = createOrderDto.UserId,
            OrderStatus = "Order Submitted"
        };
        await repository.CreateAsync(order);

+         //async validate
+         // var endpoint = await sendEndpointProvider.GetSendEndpoint(
+         //     new Uri("queue:product-validate-product")
+         // );
+         // await endpoint.Send(new ValidateProduct{
+         //     OrderId = order.Id,
+         //     ProductId = createOrderDto.ProductId
+         // });
+ 
+         //async Orchestrator
+         await publishEndpoint.Publish<OrderSubmitted>(
+             new OrderSubmitted {
+                 OrderId = order.Id,
+                 ProductId = createOrderDto.ProductId
+             });

+         await uow.SaveChangeAsync();

        return CreatedAtAction(nameof(PostAsync), order);
    }
}
```