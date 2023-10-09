# Create MSA.Common.Contracts for some common contracts shared across service

# Create MSA.Common.Contracts class library project
## Create new class library project
- via Visual studio code, run below command to create new class library project
``` bash
dotnet new classlib -n MSA.Common.Contracts
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.Common.Contracts
```

## Add common nuget package
- via Visual studio code, run below command to install common nuget package 
``` bash
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.DependencyInjection
dotnet add package Microsoft.Extensions.Configuration.Binder
```

## Create new folder named Settings
- via visual studio code, create new folder name **Settings** via MSA.Common.Contracts Project
### Create new C# file **MongoDBSetting.cs**
- Create new C# file **MongoDBSetting.cs** for mapping appSettings config to C# strong typed
``` C#
namespace MSA.Common.Contracts.Settings;

public class MongoDBSetting
{
    public string Host { get; init; }

    public string Port { get; init; }

    public string ConnectionString => $"mongodb://{Host}:{Port}";
}
```

### Create new C# file **PostgresDBSetting.cs**
- Create new C# file **PostgresDBSetting.cs** for mapping appSettings config to C# strong typed

``` C#
namespace MSA.Common.Contracts.Settings;

public class PostgresDBSetting
{
    public string ConnectionString { get; set; }
}
```

### Create new C# file **RabbitMQSetting.cs**
- Create new C# file **RabbitMQSetting.cs** for mapping appSettings config to C# strong typed

``` C#
namespace MSA.Common.Contracts.Settings;

public class RabbitMQSetting
{
    public string Host { get; init; }
}
```

### Create new C# file **ServiceUrlsSetting.cs**
- Create new C# file **ServiceUrlsSetting.cs** 

``` C#
namespace MSA.Common.Contracts.Settings;

public class ServiceUrlsSetting
{
    public string IdentityServiceUrl { get; set; }
    public string ProductServiceUrl { get; set; }
    public string OrderServiceUrl { get; set; }
}
```

### Create new C# file **ServiceEndpointSettings.cs**
- Create new C# file **ServiceEndpointSettings.cs** for mapping appSettings config to C# strong typed

``` C#
namespace MSA.Common.Contracts.Settings;

public class ServiceEndpointSettings
{
    public string BankService { get; set; }
    public string OrderService { get; set; }
    public string ProductService { get; set; }
}
```

### Create new C# file **ServiceSetting.cs**
- Create new C# file **ServiceSetting.cs** for mapping appSettings config to C# strong typed

``` C#
namespace MSA.Common.Contracts.Settings;

public class ServiceSetting
{
    public string ServiceName { get; init; }
}
```

## Create new folder named **Domain** via MSA.Common.Contracts project
- Create new folder named **Domain** via MSA.Common.Contracts project

### Create new file **IEntity.cs** under Domain folder
- Create new file **IEntity.cs** under domain folder for generic Entity type

``` C#
namespace MSA.Common.Contracts.Domain;

public interface IEntity
{
    Guid Id { get; set; }
}
```

### Create new file **IRepository.cs** under Domain folder
- Create new file **IRepository.cs** under domain folder for generic repository type

``` C#
using System.Linq.Expressions;

namespace MSA.Common.Contracts.Domain;

public interface IRepository<T> where T : IEntity
{
    Task CreateAsync(T entity);
    Task DeleteAsync(T entity);
    Task<IReadOnlyCollection<T>> GetAllAsync();
    Task<IReadOnlyCollection<T>> GetAllAsync(Expression<Func<T, bool>> filter);
    Task<T> GetAsync(Guid id);
    Task<T> GetAsync(Expression<Func<T, bool>> filter);
    Task UpdateAsync(T entity);
}
```

## Verify the library class is built successfully
- via visual studio code, run below command to ensure project is built successfully

``` bash
dotnet build
```