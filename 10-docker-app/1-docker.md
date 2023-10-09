# Dockerize identity, reverse proxy and product service

## Dockerize Product Service
- Create file **update-ca-certificate-product.sh** via folder **aspnet/https** and paste below content to it
``` sh
#!/bin/sh

echo "startup script is running"

cp /app/https/localhost.crt /usr/local/share/ca-certificates
update-ca-certificates

dotnet MSA.ProductService.dll

```

- From MSA.ProductService folder, create new **Dockerfile** file and paste below content to it
``` Dockerfile
FROM mcr.microsoft.com/dotnet/nightly/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 5002
EXPOSE 15002

RUN mkdir -p startup
COPY ./aspnet/https/update-ca-certificate-product.sh /startup/
RUN chmod +x /startup/update-ca-certificate-product.sh

FROM mcr.microsoft.com/dotnet/sdk:7.0-jammy AS build
WORKDIR /src
COPY ["./MSA.ProductService/MSA.ProductService.csproj","src/MSA.ProductService/"]
COPY ["./MSA.Common.Contracts/MSA.Common.Contracts.csproj","src/MSA.Common.Contracts/"]
COPY ["./MSA.Common.Mongo/MSA.Common.Mongo.csproj","src/MSA.Common.Mongo/"]
COPY ["./MSA.Common.PostgresMassTransit/MSA.Common.PostgresMassTransit.csproj","src/MSA.Common.PostgresMassTransit/"]
COPY ["./MSA.Common.Security/MSA.Common.Security.csproj","src/MSA.Common.Security/"]
RUN dotnet restore "src/MSA.ProductService/MSA.ProductService.csproj"
COPY . .
WORKDIR "/src/MSA.ProductService"
RUN dotnet build "MSA.ProductService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MSA.ProductService.csproj" -c Release -o /app/publish --self-contained false --no-restore

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["/startup/update-ca-certificate-product.sh"]
```

- Open terminal from **src** folder and run below command. Ensure it built successfully
``` bash
docker build . -f ./MSA.ProductService/Dockerfile -t product-service:latest
```

## Dockerize identity server
- Create file **update-ca-certificate-identity.sh** via folder **aspnet/https** and paste below content to it
``` sh
#!/bin/sh

echo "startup script is running"

cp /app/https/localhost.crt /usr/local/share/ca-certificates
update-ca-certificates
dotnet MSA.IdentityServer.dll

```

- From MSA.IdentityServer folder, create new **Dockerfile** file and paste below content to it
``` Dockerfile
FROM mcr.microsoft.com/dotnet/nightly/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 5001
EXPOSE 15001

RUN mkdir -p startup
COPY ./aspnet/https/update-ca-certificate-identity.sh /startup/
RUN chmod +x /startup/update-ca-certificate-identity.sh

FROM mcr.microsoft.com/dotnet/sdk:7.0-jammy AS build
WORKDIR /src
COPY ["./MSA.IdentityServer/MSA.IdentityServer.csproj","src/MSA.IdentityServer/"]
RUN dotnet restore "src/MSA.IdentityServer/MSA.IdentityServer.csproj"
COPY . .
WORKDIR "/src/MSA.IdentityServer"
RUN dotnet build "MSA.IdentityServer.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MSA.IdentityServer.csproj" -c Release -o /app/publish --self-contained false --no-restore

FROM base AS final
WORKDIR /app

COPY --from=publish /app/publish .
ENTRYPOINT ["/startup/update-ca-certificate-identity.sh"]
```

- Modify MSA.Common.Security\Authentication\Extension.cs for temporary resolve the issue **RemoteCertificateNameMismatch** when access the wellknown discovery link of duende identity server due to using local self-signed certificate
``` Diff
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

+                options.BackchannelHttpHandler = new HttpClientHandler()
+                {
+                    ServerCertificateCustomValidationCallback = HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
+                };

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
```


- Open terminal from **src** folder and run below command. Ensure it built successfully
``` bash
docker build . -f ./MSA.IdentityServer/Dockerfile -t identity-service:latest
```

## Dockerize Reverse Proxy server
- Create file **update-ca-certificate-reverseproxy.sh** via folder **aspnet/https** and paste below content to it
``` sh
#!/bin/sh

echo "startup script is running"

cp /app/https/localhost.crt /usr/local/share/ca-certificates
update-ca-certificates

dotnet MSA.ReverseProxy.dll

```

- From MSA.ReverseProxy folder, create new **Dockerfile** file and paste below content to it
``` Dockerfile
FROM mcr.microsoft.com/dotnet/nightly/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 18080

RUN mkdir -p startup
COPY ./aspnet/https/update-ca-certificate-reverseproxy.sh /startup/
RUN chmod +x /startup/update-ca-certificate-reverseproxy.sh

FROM mcr.microsoft.com/dotnet/sdk:7.0-jammy AS build
WORKDIR /src
COPY ["./MSA.ReverseProxy/MSA.ReverseProxy.csproj","src/MSA.ReverseProxy/"]
RUN dotnet restore "src/MSA.ReverseProxy/MSA.ReverseProxy.csproj"
COPY . .
WORKDIR "/src/MSA.ReverseProxy"
RUN dotnet build "MSA.ReverseProxy.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MSA.ReverseProxy.csproj" -c Release -o /app/publish --self-contained false --no-restore

FROM base AS final
WORKDIR /app

COPY --from=publish /app/publish .
ENTRYPOINT ["/startup/update-ca-certificate-reverseproxy.sh"]
```

- Open terminal from **src** folder and run below command. Ensure it built successfully
``` bash
docker build . -f ./MSA.ReverseProxy/Dockerfile -t reverse-proxy:latest
```

## Add those 3 service to docker-compose file
- Replace docker-compose.yaml content as below
``` yaml
version: "3.8"

services:
  mongo:
    image: mongo
    container_name: mongo
    ports:
      - 27017:27017
    volumes:
      - mongodbdata:/data/db
    networks:
      - msa-network

  rabbitmq:
    image: rabbitmq:management
    container_name: rabbitmq
    ports:
      - 5672:5672
      - 15672:15672
    volumes:
      - rabbitmqdata:/var/lib/rabbitmq
    hostname: rabbitmq
    networks:
      - msa-network

  postgresql:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=msapostgres
      - POSTGRES_USER=guest
      - POSTGRES_PASSWORD=guest
    ports:
      - 5432:5432
    volumes:
      - postgresqldata:/var/lib/postgresql/data
    networks:
      - msa-network

  product-service:
    image: product-service:latest
    build:
      context: .
      dockerfile: ./MSA.ProductService/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - MongoDBSetting__Host=mongo
      - ASPNETCORE_URLS=https://+:5002
      - RabbitMQSetting__Host=rabbitmq
      - ServiceUrlsSetting__IdentityServiceUrl=https://identity-service:5001
      - Kestrel__Certificates__Default__Password=MsaFundamental
      - Kestrel__Certificates__Default__Path=https/localhost.pfx
    volumes:
      - ./aspnet/https:/app/https:ro
      - /usr/local/share/ca-certificates:/usr/local/share/ca-certificates
      - /etc/ssl/certs:/etc/ssl/certs
    ports:
      - "5002:5002"
    restart: always
    networks:
      - msa-network

  identity-service:
    image: identity-service:latest
    build:
      context: .
      dockerfile: ./MSA.IdentityServer/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:5001;http://+:15001
      - Kestrel__Certificates__Default__Password=MsaFundamental
      - Kestrel__Certificates__Default__Path=https/localhost.pfx
    volumes:
      - ./aspnet/https:/app/https:ro
      - /usr/local/share/ca-certificates:/usr/local/share/ca-certificates
      - /etc/ssl/certs:/etc/ssl/certs
    ports:
      - "5001:5001"
      - "15001:15001"
    restart: always
    networks:
      - msa-network

  reverse-proxy:
    image: reverse-proxy:latest
    build:
      context: .
      dockerfile: ./MSA.ReverseProxy/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ReverseProxy__Clusters__identityApiCluster__Destinations__destination1__Address=https://identity-service:5001/
      - ReverseProxy__Clusters__productApiCluster__Destinations__destination1__Address=https://product-service:5002/
      - ASPNETCORE_URLS=https://+:8080;http://+:18080
      - Kestrel__Certificates__Default__Password=MsaFundamental
      - Kestrel__Certificates__Default__Path=https/localhost.pfx
    volumes:
      - ./aspnet/https:/app/https:ro
      - /usr/local/share/ca-certificates:/usr/local/share/ca-certificates
      - /etc/ssl/certs:/etc/ssl/certs
    ports:
      - "8080:8080"
      - "18080:18080"
    restart: always
    networks:
      - msa-network

volumes:
  mongodbdata:
  rabbitmqdata:
  postgresqldata:

networks:
  msa-network:
```

## Run docker compose and test to ensure all are working well
- use below command to run all the services
``` bash
docker compose up
```

- use the same file via **REST Client/reverse-proxy/http** and ensure all endpoint are working correctly
