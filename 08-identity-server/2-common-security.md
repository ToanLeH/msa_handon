# Create MSA.Common.Security for handle authentication

# Create MSA.Common.Security class library project
## Create new class library project
- via Visual studio code, run below command to create new class library project
``` bash
dotnet new classlib -n MSA.Common.Security
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.Common.Security
```

## Install Nuget package 
- via Visual studio code, run below command to install mongo nuget package
``` bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

## Add reference to MSA.Common.Contracts project
- via Visual studio code, run below command to add reference to MSA.Common.Contracts
``` bash
dotnet add reference ../MSA.Common.Contracts
```

## Create folder **Authentication** via MSA.Common.Security project
- Create folder **Authentication** via MSA.Common.Security project
- Create **Extension.cs** file via **Authentication** folder and paste below code to it
``` C#
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MSA.Common.Contracts.Settings;

namespace MSA.Common.Security.Authentication;

public static class Extensions 
{
    public static IServiceCollection AddMSAAuthentication(this IServiceCollection services)
    {
        services
            .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                var srvProvider = services.BuildServiceProvider();
                var config = srvProvider.GetService<IConfiguration>();
                var srvUrlsSetting = config.GetSection(nameof(ServiceUrlsSetting)).Get<ServiceUrlsSetting>();

                options.Authority = srvUrlsSetting.IdentityServiceUrl;
                options.RequireHttpsMetadata = false;

                options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidIssuers = new List<string>
                    {
                        "https://identity-api:5001",
                        "https://localhost:5001",
                        "https://localhost:8080"
                    },
                    ValidAudiences = new List<string>
                    {
                        "https://identity-api:5001/resources",
                        "https://localhost:5001/resources",
                        "productapi",
                        "orderapi"
                    }
                };
            });
        return services;
    }
}
```

## Create folder **Authorization** via MSA.Common.Security project
- Create folder **Authorization** via MSA.Common.Security project
- Create **Extension.cs** file via **Authorization** folder and paste below code to it
``` C#
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.DependencyInjection;

namespace MSA.Common.Security.Authorization;

public static class Extensions 
{
    public static IServiceCollection AddMSAAuthorization(
        this IServiceCollection services,
        Action<AuthorizationOptions> configure)
    {
        services.AddAuthorization(configure);

        return services;
    }
}
```

### Build projcet
- Run below command to build project and ensure there's no errors
``` bash
dotnet build
```