# Create MSA.Common.MongoDB for integrating with MongoDB shared across service

# Create MSA.Common.MongoDB class library project
## Create new class library project
- via Visual studio code, run below command to create new class library project
``` bash
dotnet new classlib -n MSA.Common.Mongo
```

## Add class library to solution
- via Visual studio code, run below command to add project to solution
``` bash
dotnet sln add MSA.Common.Mongo
```

## Add reference to MSA.Common.Contracts project
- via Visual studio code, run below command to add reference to MSA.Common.Contracts
``` bash
dotnet add reference ../MSA.Common.Contracts
```

## Install Nuget package relating to Mongo integration
- via Visual studio code, run below command to install mongo nuget package
``` bash
dotnet add package MongoDB.Driver
```

## Create new file **Extensions.cs**
- via visual studio code, create new file **Enxtensions.cs** and add below code to it

``` C#
using MongoDB.Bson;
using MongoDB.Bson.Serialization;
using MongoDB.Bson.Serialization.Serializers;
using MongoDB.Driver;
using MSA.Common.Contracts.Settings;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;
using MSA.Common.Contracts.Domain;

namespace MSA.Common.MongoDB;

public static class Extensions 
{
    public static IServiceCollection AddMongo(this IServiceCollection services)
    {
        BsonSerializer.RegisterSerializer(new GuidSerializer(BsonType.String));
        BsonSerializer.RegisterSerializer(new DateTimeOffsetSerializer(BsonType.String));

        //Register Mongo Client
        services.AddSingleton(ServiceProvider => {
            var configuration = ServiceProvider.GetService<IConfiguration>();
            var serviceSetting = configuration.GetSection(nameof(ServiceSetting)).Get<ServiceSetting>();
            var mongoDBSetting = configuration.GetSection(nameof(MongoDBSetting)).Get<MongoDBSetting>();
            var mongoClient = new MongoClient(mongoDBSetting.ConnectionString);
            return mongoClient.GetDatabase(serviceSetting.ServiceName);
        });

        return services;
    }

    public static IServiceCollection AddRepositories<T>(this IServiceCollection services, string collectionName) where T : IEntity
    {
        services.AddSingleton<IRepository<T>>(serviceProvider => 
        {
            var database = serviceProvider.GetService<IMongoDatabase>();
            return new MongoRepository<T>(database, collectionName);
        });
        return services;
    }
}
```

## Create new file **MongoRepository.cs**
- via visual studio code, create new file **MongoRepository.cs** and add below code to it

``` C#
using System.Linq.Expressions;
using System.Runtime.CompilerServices;
using MongoDB.Driver;
using MSA.Common.Contracts.Domain;

namespace MSA.Common.MongoDB;

public class MongoRepository<T> : IRepository<T> where T : IEntity
{
    private readonly IMongoCollection<T> dbCollection;
    private readonly FilterDefinitionBuilder<T> filterBuilder = Builders<T>.Filter;

    public MongoRepository(IMongoDatabase database, string collectionName)
    {
        dbCollection = database.GetCollection<T>(collectionName);
    }

    public async Task<IReadOnlyCollection<T>> GetAllAsync()
    {
        return await dbCollection.Find(filterBuilder.Empty).ToListAsync();
    }

    public async Task<IReadOnlyCollection<T>> GetAllAsync(Expression<Func<T, bool>> filter)
    {
        return await dbCollection.Find(filter).ToListAsync();
    }

    public async Task<T> GetAsync(Guid id)
    {
        FilterDefinition<T> filter = filterBuilder.Eq(entity => entity.Id, id);
        return await dbCollection.Find(filter).FirstOrDefaultAsync();
    }

    public async Task<T> GetAsync(Expression<Func<T, bool>> filter)
    {
        return await dbCollection.Find(filter).FirstOrDefaultAsync();
    }

    public async Task CreateAsync(T entity)
    {
        if (entity == null)
        {
            throw new ArgumentNullException(nameof(entity));
        }
        await dbCollection.InsertOneAsync(entity);
    }

    public async Task UpdateAsync(T entity)
    {
        if (entity == null)
        {
            throw new ArgumentNullException(nameof(entity));
        }
        FilterDefinition<T> filter = filterBuilder.Eq(
            existingEntity => existingEntity.Id, entity.Id);
        await dbCollection.ReplaceOneAsync(filter, entity);
    }

    public async Task DeleteAsync(T entity)
    {
        FilterDefinition<T> filter = filterBuilder.Eq(
            existingEntity => existingEntity.Id, entity.Id);
        await dbCollection.DeleteOneAsync(filter);
    }

}
```

## Verify the library class is built successfully
- via visual studio code, run below command to ensure project is built successfully

``` bash
dotnet build
```