# Create an MSA project skeleton that supports Duende IdentityServer as an Identity Provider Using In Memory Database

# Create MSA.IdentityServer web api project
## Create new webapi project
- via Visual studio code, run below command to create new webapi project
``` bash
dotnet new webapi -n MSA.IdentityServer
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.IdentityServer
```

## Add DuenDe.IdentityServer 
- add Duende.IdentityServer package into project
``` bash
dotnet add package DuenDe.IdentityServer
```

## Add folder DummyData into project
- This folder will be used to store Test Users and Test Configurations for Duende IdentityServer.
- Right click on project and create new folder named **DummyData**
- Create **TestUsers.cs** file and paste below code to it 
``` C#
using IdentityModel;
using System.Security.Claims;
using System.Text.Json;
using Duende.IdentityServer.Test;
using Duende.IdentityServer;

namespace MSA.IdentityService.DummyData
{
    public class TestUsers
    {
        public static List<TestUser> Users
        {
            get
            {
                var address = new
                {
                    street_address = "One Hacker Way",
                    locality = "Heidelberg",
                    postal_code = 69118,
                    country = "Germany"
                };
                
                return new List<TestUser>
                {
                    new TestUser
                    {
                        SubjectId = "818727",
                        Username = "alice",
                        Password = "alice",
                        Claims =
                        {
                            new Claim(JwtClaimTypes.Name, "Alice Smith"),
                            new Claim(JwtClaimTypes.GivenName, "Alice"),
                            new Claim(JwtClaimTypes.FamilyName, "Smith"),
                            new Claim(JwtClaimTypes.Email, "AliceSmith@email.com"),
                            new Claim(JwtClaimTypes.EmailVerified, "true", ClaimValueTypes.Boolean),
                            new Claim(JwtClaimTypes.WebSite, "http://alice.com"),
                            new Claim(JwtClaimTypes.Address, JsonSerializer.Serialize(address), IdentityServerConstants.ClaimValueTypes.Json)
                        }
                    },
                    new TestUser
                    {
                        SubjectId = "88421113",
                        Username = "bob",
                        Password = "bob",
                        Claims =
                        {
                            new Claim(JwtClaimTypes.Name, "Bob Smith"),
                            new Claim(JwtClaimTypes.GivenName, "Bob"),
                            new Claim(JwtClaimTypes.FamilyName, "Smith"),
                            new Claim(JwtClaimTypes.Email, "BobSmith@email.com"),
                            new Claim(JwtClaimTypes.EmailVerified, "true", ClaimValueTypes.Boolean),
                            new Claim(JwtClaimTypes.WebSite, "http://bob.com"),
                            new Claim(JwtClaimTypes.Address, JsonSerializer.Serialize(address), IdentityServerConstants.ClaimValueTypes.Json)
                        }
                    }
                };
            }
        }
    }
}
```

- Create **Config.cs** and paste below code to it
``` C#
using Duende.IdentityServer.Models;

namespace MSA.IdentityService.DummyData
{
    public static class Config
    {
        public static IEnumerable<IdentityResource> IdentityResources =>
            new IdentityResource[]
            {
                new IdentityResources.OpenId(),
                new IdentityResources.Profile(),
            };

        public static IEnumerable<ApiScope> ApiScopes =>
            new ApiScope[]
            {
                new ApiScope("productapi.read"),
                new ApiScope("productapi.write"),
                new ApiScope("orderapi.read"),
                new ApiScope("orderapi.write"),
            };

        public static IEnumerable<ApiResource> ApiResources => new[]
        {
            new ApiResource("productapi")
            {
                Scopes = new List<string> {"productapi.read", "productapi.write"},
                ApiSecrets = new List<Secret> { new Secret("Scopesecret".Sha256())},
                UserClaims = new List<string> {"role"}
            },
            new ApiResource("orderapi")
            {
                Scopes = new List<string> {"orderapi.read", "orderapi.write"},
                ApiSecrets = new List<Secret> { new Secret("Scopesecret".Sha256())},
                UserClaims = new List<string> {"role"}
            }
        };
            
        public static IEnumerable<Client> Clients =>
            new Client[]
            {
                // m2m client credentials flow client
                new Client
                {
                    ClientId = "m2m.client",
                    ClientName = "Client Credentials Client",

                    AllowedGrantTypes = GrantTypes.ClientCredentials,
                    ClientSecrets = { new Secret("511536EF-F270-4058-80CA-1C89C192F69A".Sha256()) },

                    AllowedScopes = { 
                        "productapi.read", 
                        "productapi.write",
                        "orderapi.read",
                        "orderapi.write" 
                    }
                },

                // interactive client using code flow + pkce
                new Client
                {
                    ClientId = "interactive",
                    ClientSecrets = { new Secret("49C1A7E1-0C79-4A89-A3D6-A37998FB86B0".Sha256()) },

                    AllowedGrantTypes = GrantTypes.Code,

                    RedirectUris = { "https://localhost:5002/signin-oidc" },
                    FrontChannelLogoutUri = "https://localhost:5002/signout-oidc",
                    PostLogoutRedirectUris = { "https://localhost:5002/signout-callback-oidc" },

                    AllowOfflineAccess = true,
                    AllowedScopes = { "openid", "profile", "productapi.read", "productapi.write" }
                },

                // product-swagger client using code flow + pkce
                new Client
                {
                    ClientId = "product-swagger",
                    RequireClientSecret = false,

                    AllowedGrantTypes = GrantTypes.Code,
                    RequirePkce = false,

                    RedirectUris = { "https://localhost:5002/swagger/oauth2-redirect.html" },
                    AllowedCorsOrigins = { "https://localhost:5002" },

                    AllowOfflineAccess = true,
                    AllowedScopes = { "openid", "profile", "productapi.read", "productapi.write" }
                }
            };
    }
}
```

## Modify *Program.cs** with below changes to integrate with Duende IdentityServer
- Add below namespace and config for identity server

``` Diff
+ using Duende.IdentityServer.Models;
+ using Duende.IdentityServer.Test;
+ using MSA.IdentityService.DummyData;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
+ builder.Services.AddIdentityServer(options => {
+     options.Events.RaiseErrorEvents = true;
+     options.Events.RaiseInformationEvents = true;
+     options.Events.RaiseSuccessEvents = true;
+     options.Events.RaiseFailureEvents = true;
+ 
+    options.EmitStaticAudienceClaim = true;
+ }).AddTestUsers(TestUsers.Users) 
+   .AddInMemoryClients(Config.Clients)
+   .AddInMemoryApiResources(Config.ApiResources)
+   .AddInMemoryApiScopes(Config.ApiScopes)
+   .AddInMemoryIdentityResources(Config.IdentityResources);

builder.Services.AddControllers();
```

- add below line to use identity server
``` Diff
// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

+ app.UseIdentityServer();

app.UseAuthorization();
```

## Modify **LaunchSettings.json** to run the project via port 5001
``` Diff
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5001;http://localhost:15001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
+      "applicationUrl": "https://localhost:5001;http://localhost:15001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
```

## Add sample UI for Duende Identity
- Install Duende Identity Templates using below command
``` bash
dotnet new install Duende.IdentityServer.Templates
```

- Add quick start ui for duende using below commmand
``` bash
dotnet new isui
```

- Modify **program.cs** to add RazorPages
``` Diff
using Duende.IdentityServer.Models;
using Duende.IdentityServer.Test;
using MSA.IdentityService.DummyData;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddIdentityServer(options => {
    options.Events.RaiseErrorEvents = true;
    options.Events.RaiseInformationEvents = true;
    options.Events.RaiseSuccessEvents = true;
    options.Events.RaiseFailureEvents = true;

   options.EmitStaticAudienceClaim = true;
}).AddTestUsers(TestUsers.Users) 
  .AddInMemoryClients(Config.Clients)
  .AddInMemoryApiResources(Config.ApiResources)
  .AddInMemoryApiScopes(Config.ApiScopes)
  .AddInMemoryIdentityResources(Config.IdentityResources);

+ builder.Services.AddRazorPages();
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
+ app.UseStaticFiles();

app.UseIdentityServer();

app.UseAuthorization();

app.MapControllers();

+ app.MapRazorPages();

app.Run();

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
- Browse to url https://localhost:5001/.well-known/openid-configuration