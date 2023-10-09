# Create an MSA project skeleton that act as Reverse Proxy using YARP

# Create MSA.ReverseProxy web api project
## Create new webapi project
- via Visual studio code, run below command to create new webapi project
``` bash
dotnet new webapi -n MSA.ReverseProxy
```

## Add Yarp.ReverseProxy
- add Yarp.ReverseProxy package into project
``` bash
dotnet add package Yarp.ReverseProxy
```

## replace content of appsettings.json to add rules
- replace content of appsettings.json as below
``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ReverseProxy": {
    "Routes": {
      "identityApiRoute" : {
        "ClusterId": "identityApiCluster",
        "Match": {
          "Path": "identity-api/{**remainder}"
        },
        "Transforms": [
          { "PathRemovePrefix": "identity-api" },
          { "PathPrefix": "/" },
          { "RequestHeaderOriginalHost": "true" }
        ]
      },
      "productApiRoute" : {
        "ClusterId": "productApiCluster",
        "Match": {
          "Path": "product-api/{**remainder}"
        },
        "Transforms": [
          { "PathRemovePrefix": "product-api" },
          { "PathPrefix": "/" },
          { "RequestHeaderOriginalHost": "true" }
        ]
      }
    },
    "Clusters": {
      "identityApiCluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://localhost:5001/"
          }
        }
      },
      "productApiCluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://localhost:5002/"
          }
        }
      }
    }
  }
}

```

## Modify **program.cs** to integrate with YARP
- Modify **program.cs** ot integrate with YARP as below
``` Diff
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
+ builder.Services.AddReverseProxy()
+    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

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

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

+ app.MapReverseProxy();

app.Run();

```

## Modify **LaunchSettings.json** to run the project via port 8080
``` Diff
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:8080;http://localhost:18080",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:8080;http://localhost:18080",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
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