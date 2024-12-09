# Building 'eShop' from Zero to Hero: Adding Basket.API project

This sample scope of work is focus on adding the **Basket Service** in the **eShop** solution

We can review the architecture picture

![image](https://github.com/user-attachments/assets/bb063b3b-d46a-4d5b-ac67-9010d48c4589)

## 1. We Download the Solution from Github repo

The starting point for this sample is based on the following github repo:

https://github.com/luiscoco/eShop_Tutorial-Step7_Provision_AI_in_AppHost

## 2. We Rename the Solution

![image](https://github.com/user-attachments/assets/bac06caa-d08a-4c1f-98d2-33efb163c830)

## 3. We Add a New Project Basket.API

We right click on the Solution name and select the menu option **Add New Project**

![image](https://github.com/user-attachments/assets/e05cea1f-c0ed-4deb-a243-3db983cd9119)

We select the **ASP.NET Core gRPC Service** project template and press on the Next button

![image](https://github.com/user-attachments/assets/39eee51b-1704-4794-8bc6-d1a4fe88af2a)

We input the project name and location and press on the Next button

![image](https://github.com/user-attachments/assets/113b03dd-1d86-45dc-883f-209ef686db8f)

We select the **.NET 9** Framework and press the Create button

![image](https://github.com/user-attachments/assets/0f2a79d4-7fe4-43e7-a520-eeafdf7a3909)

## 4. We Load the Nuget packages and we create the GlobalUsings.cs file (Basket.API project)

![image](https://github.com/user-attachments/assets/59ce7bb9-8ea3-411d-a3f7-1e384594062a)

**Aspire.StackExchange.Redis**: This package is typically an extension or a customization of the popular StackExchange.Redis library, which is widely used for working with Redis, a high-performance in-memory data store

Use Cases:

Provides advanced features or abstractions for integrating Redis into .NET applications

Likely tailored for scenarios requiring distributed caching, pub/sub messaging, or session storage with Redis

May include additional features like connection management, serialization helpers, or specific configurations for easier integration with Redis in Aspire-related projects

Dependencies: It builds upon StackExchange.Redis, which is the primary library for Redis in .NET

**Grpc.AspNetCore**: This package provides tools for building gRPC services in ASP.NET Core applications

Use Cases:

Enables server and client implementations of gRPC (Google Remote Procedure Call), which is a high-performance, open-source, and language-neutral framework for remote procedure calls

Facilitates communication between microservices in distributed systems

Provides support for HTTP/2, which is essential for gRPC communication

Includes middleware and tools for defining, hosting, and consuming gRPC services in ASP.NET Core

Dependencies: It integrates tightly with the ASP.NET Core pipeline and leverages the Protobuf serialization format for defining service contracts and data exchange

## 5. We Delete Protos and Services folders

## 6. We Create the Folders structure (Basket.API project)

![image](https://github.com/user-attachments/assets/02c6f4aa-aada-4e31-a415-e4114e2d1e1b)

## 7. We Create the Data Model (Basket.API project)

We have to add two new files for defining the data model:

![image](https://github.com/user-attachments/assets/cec328e4-9750-473c-a7ca-8fbaa012ca66)

**BasketItem** represents an item in a shopping basket, including properties to describe the product, its pricing, quantity, and picture URL

It implements the **IValidatableObject** interface to include custom validation logic

This class is be used in an **e-commerce** system's basket management component

Before processing or persisting **BasketItem** objects, **validation** ensures that they meet the required **business rules** (e.g., Quantity must be positive)

If a user attempts to add a basket item with a Quantity of 0, the Validate method would catch the issue and return a validation error: "Invalid number of units"

This helps maintain consistent and valid data across the application

**BasketItem.cs**

```csharp
using System.ComponentModel.DataAnnotations;

namespace eShop.Basket.API.Model;

public class BasketItem : IValidatableObject
{
    public string Id { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal OldUnitPrice { get; set; }
    public int Quantity { get; set; }
    public string PictureUrl { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        var results = new List<ValidationResult>();

        if (Quantity < 1)
        {
            results.Add(new ValidationResult("Invalid number of units", new[] { "Quantity" }));
        }

        return results;
    }
}
```

The **CustomerBasket** class represents a shopping basket associated with a specific customer

It contains the customer ID and a collection of items in the basket

**CustomerBasket.cs**

```csharp
namespace eShop.Basket.API.Model;

public class CustomerBasket
{
    public string BuyerId { get; set; }

    public List<BasketItem> Items { get; set; } = [];

    public CustomerBasket() { }

    public CustomerBasket(string customerId)
    {
        BuyerId = customerId;
    }
}
```

## 8. We Create Redis Repository (Basket.API project)

We have to add two files for defining the Repository:

![image](https://github.com/user-attachments/assets/85697374-df0e-456c-9fa2-ca9d16561416)

The interface outlines the contract for a repository that **manages** operations related to **customer baskets** (shopping carts)

**IBasketRepository.cs**

```csharp
using eShop.Basket.API.Model;

namespace eShop.Basket.API.Repositories;

public interface IBasketRepository
{
    Task<CustomerBasket> GetBasketAsync(string customerId);
    Task<CustomerBasket> UpdateBasketAsync(CustomerBasket basket);
    Task<bool> DeleteBasketAsync(string id);
}
```

This C# code represents a repository implementation for **managing customer baskets** in a **Redis**-backed e-commerce application

This implementation is an efficient and scalable way to manage user-specific data in Redis, leveraging modern C# features like source generators and memory-efficient APIs for serialization and deserialization

**Redis** is chosen for storing baskets in e-commerce applications because it offers unmatched speed, simplicity, and scalability for handling ephemeral, high-traffic data. These qualities align well with the dynamic nature of shopping baskets, ensuring users experience real-time responsiveness and reliability



**RedisBasketRepository.cs**

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using eShop.Basket.API.Model;
using StackExchange.Redis;

namespace eShop.Basket.API.Repositories;

public class RedisBasketRepository(ILogger<RedisBasketRepository> logger, IConnectionMultiplexer redis) : IBasketRepository
{
    private readonly IDatabase _database = redis.GetDatabase();

    // implementation:

    // - /basket/{id} "string" per unique basket
    private static RedisKey BasketKeyPrefix = "/basket/"u8.ToArray();
    // note on UTF8 here: library limitation (to be fixed) - prefixes are more efficient as blobs

    private static RedisKey GetBasketKey(string userId) => BasketKeyPrefix.Append(userId);

    public async Task<bool> DeleteBasketAsync(string id)
    {
        return await _database.KeyDeleteAsync(GetBasketKey(id));
    }

    public async Task<CustomerBasket> GetBasketAsync(string customerId)
    {
        using var data = await _database.StringGetLeaseAsync(GetBasketKey(customerId));

        if (data is null || data.Length == 0)
        {
            return null;
        }
        return JsonSerializer.Deserialize(data.Span, BasketSerializationContext.Default.CustomerBasket);
    }

    public async Task<CustomerBasket> UpdateBasketAsync(CustomerBasket basket)
    {
        var json = JsonSerializer.SerializeToUtf8Bytes(basket, BasketSerializationContext.Default.CustomerBasket);
        var created = await _database.StringSetAsync(GetBasketKey(basket.BuyerId), json);

        if (!created)
        {
            logger.LogInformation("Problem occurred persisting the item.");
            return null;
        }


        logger.LogInformation("Basket item persisted successfully.");
        return await GetBasketAsync(basket.BuyerId);
    }
}

[JsonSerializable(typeof(CustomerBasket))]
[JsonSourceGenerationOptions(PropertyNameCaseInsensitive = true)]
public partial class BasketSerializationContext : JsonSerializerContext
{

}
```

## 9. We Create the Service and Proto file (Basket.API project)

We create a new Service file **BasketService.cs**

![image](https://github.com/user-attachments/assets/7d328efa-8c97-41c9-be1d-df6bc5891d2a)

We review the **BasketService.cs**

```csharp
using System.Diagnostics.CodeAnalysis;
using eShop.Basket.API.Repositories;
using eShop.Basket.API.Extensions;
using eShop.Basket.API.Model;
using Microsoft.AspNetCore.Authorization;
using Grpc.Core;

namespace eShop.Basket.API.Grpc;

public class BasketService(
    IBasketRepository repository,
    ILogger<BasketService> logger) : Basket.BasketBase
{
    [AllowAnonymous]
    public override async Task<CustomerBasketResponse> GetBasket(GetBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            return new();
        }

        if (logger.IsEnabled(LogLevel.Debug))
        {
            logger.LogDebug("Begin GetBasketById call from method {Method} for basket id {Id}", context.Method, userId);
        }

        var data = await repository.GetBasketAsync(userId);

        if (data is not null)
        {
            return MapToCustomerBasketResponse(data);
        }

        return new();
    }

    public override async Task<CustomerBasketResponse> UpdateBasket(UpdateBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            ThrowNotAuthenticated();
        }

        if (logger.IsEnabled(LogLevel.Debug))
        {
            logger.LogDebug("Begin UpdateBasket call from method {Method} for basket id {Id}", context.Method, userId);
        }

        var customerBasket = MapToCustomerBasket(userId, request);
        var response = await repository.UpdateBasketAsync(customerBasket);
        if (response is null)
        {
            ThrowBasketDoesNotExist(userId);
        }

        return MapToCustomerBasketResponse(response);
    }

    public override async Task<DeleteBasketResponse> DeleteBasket(DeleteBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            ThrowNotAuthenticated();
        }

        await repository.DeleteBasketAsync(userId);
        return new();
    }

    [DoesNotReturn]
    private static void ThrowNotAuthenticated() => throw new RpcException(new Status(StatusCode.Unauthenticated, "The caller is not authenticated."));

    [DoesNotReturn]
    private static void ThrowBasketDoesNotExist(string userId) => throw new RpcException(new Status(StatusCode.NotFound, $"Basket with buyer id {userId} does not exist"));

    private static CustomerBasketResponse MapToCustomerBasketResponse(CustomerBasket customerBasket)
    {
        var response = new CustomerBasketResponse();

        foreach (var item in customerBasket.Items)
        {
            response.Items.Add(new BasketItem()
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
            });
        }

        return response;
    }

    private static CustomerBasket MapToCustomerBasket(string userId, UpdateBasketRequest customerBasketRequest)
    {
        var response = new CustomerBasket
        {
            BuyerId = userId
        };

        foreach (var item in customerBasketRequest.Items)
        {
            response.Items.Add(new()
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
            });
        }

        return response;
    }
}
```

We also add the proto file **basket.proto**

![image](https://github.com/user-attachments/assets/e2fc179b-98b4-475b-8144-2b0d755c7e34)

We have to right click on the **Proto** folder and add a new **basket.proto** file

![image](https://github.com/user-attachments/assets/e9f0fff0-184f-4e3e-b8f0-2492a7bdf715)

![image](https://github.com/user-attachments/assets/6a9cb154-c1ed-45ca-948d-910f00874af6)

We have to configure the Proto file properties

![image](https://github.com/user-attachments/assets/17e6cd5b-de8c-48db-9989-30dbce3f66b9)

We also review the **basket.proto** source code:

```csharp
syntax = "proto3";

option csharp_namespace = "eShop.Basket.API.Grpc";

package BasketApi;

service Basket {
    rpc GetBasket(GetBasketRequest) returns (CustomerBasketResponse) {}
    rpc UpdateBasket(UpdateBasketRequest) returns (CustomerBasketResponse) {}
    rpc DeleteBasket(DeleteBasketRequest) returns (DeleteBasketResponse) {}
}

message GetBasketRequest {
}

message CustomerBasketResponse {
    repeated BasketItem items = 1;
}

message BasketItem {
    int32 product_id = 2;
    int32 quantity = 6;
}

message UpdateBasketRequest {
    repeated BasketItem items = 2;
}

message DeleteBasketRequest {
}

message DeleteBasketResponse {
}
```

## 10. We Define the Middleware and the Extensions files (Basket.API project)

We have to add the Extensions files: **Extensions.cs** and **ServerCallContextIdentityExtensions.cs**

![image](https://github.com/user-attachments/assets/ad7d1cb1-bd45-4f7e-a2e1-3b5de6c7e4b5)

We review the  **Extensions.cs** source code:

```csharp
using System.Text.Json.Serialization;
using eShop.Basket.API.Repositories;
using eShop.ServiceDefaults;

namespace eShop.Basket.API.Extensions;

public static class Extensions
{
    public static void AddApplicationServices(this IHostApplicationBuilder builder)
    {
        builder.AddDefaultAuthentication();

        builder.AddRedisClient("redis");

        builder.Services.AddSingleton<IBasketRepository, RedisBasketRepository>();
    }
}
```

We also review the **ServerCallContextIdentityExtensions.cs** code:

```csharp
#nullable enable
using Grpc.Core;
namespace eShop.Basket.API.Extensions;

internal static class ServerCallContextIdentityExtensions
{
    public static string? GetUserIdentity(this ServerCallContext context) => context.GetHttpContext().User.FindFirst("sub")?.Value;
    public static string? GetUserName(this ServerCallContext context) => context.GetHttpContext().User.FindFirst(x => x.Type == ClaimTypes.Name)?.Value;
}
```

We also define the middleware

![image](https://github.com/user-attachments/assets/731c9275-62ff-4286-849e-58b01f94cbf0)

**Program.cs**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.AddApplicationServices();

builder.Services.AddGrpc();

var app = builder.Build();

app.MapDefaultEndpoints();

app.MapGrpcService<BasketService>();

app.Run();
```

## 11. We add orchestrator support in the Basket.API project

We right click on the Basket.API project and we select the menu option **Add .NET Aspire Orchestrator Support...**

![image](https://github.com/user-attachments/assets/22e5d264-e097-4ef6-bab0-cfcb2c01c79d)

We confirm we also include the **eShop.ServicesDefault** project as a new refernce in the **Basket.API** project

![image](https://github.com/user-attachments/assets/fdc0053b-f9d0-4004-9689-12f696f85e67)

We right click on the Basket.API project and Set As StartUp project

We **build** the **Basket.API** project for generating the Proto files

![image](https://github.com/user-attachments/assets/674b5326-f271-41c8-a70e-55d6eb919618)

We can also review the generated code

![image](https://github.com/user-attachments/assets/8dfff05a-4d63-41e8-944e-70d9a3b9fb4f)

## 12. We confirm the Basket.API project reference was included in the eShop.AppHost project

![image](https://github.com/user-attachments/assets/baa12a59-1959-4565-ae05-0aed7bd75a2e)

## 13. We Add the Nuget packages (eShop.AppHost project)

![image](https://github.com/user-attachments/assets/d6a77ab3-95d9-4f66-90fc-08aa0626f6b8)

**Aspire.Hosting.Redis**: this NuGet package provides extension methods and resource definitions for configuring a Redis resource within a .NET Aspire AppHost

This integration enables seamless setup and management of Redis instances in your distributed applications

## 14. We Modify the appsettings.json (Basket.API project)

In authentication or token-based systems (like OAuth or JWT), the **audience** often refers to the entity (service or application) that the token is intended for

The value "**basket**" could signify a specific service, such as an **e-commerce basket service** or API, indicating that this configuration is relevant for it

We modify the **appsettings.json** for adding this code:

```json
  "Identity": {
    "Audience": "basket"
  }
```

## 15. We Modify the Middleware (eShop.AppHost project)

We add the **Redis** service reference

```csharp
var redis = builder.AddRedis("redis");
```

We also have to add the **Basket.API** service registration in the middleware

```csharp
var basketApi = builder.AddProject<Projects.Basket_API>("basket-api")
    .WithReference(redis)
    .WithEnvironment("Identity__Url", identityEndpoint);
```

We add also the **Basket.API** reference in the **WebAp**

```csharp
var webApp = builder.AddProject<Projects.WebApp>("webapp", launchProfileName)
    .WithExternalHttpEndpoints()
    .WithReference(basketApi)
    .WithReference(catalogApi)
    .WithEnvironment("IdentityUrl", identityEndpoint);
```

## 16. We Add the Cart Icon (WebApp project)

We have to add the cart svg image in the wwwroot folder

![image](https://github.com/user-attachments/assets/604eb28e-1b8b-460f-963a-4abff8f0e9f0)

The cart image will be used in the CartMenu.razor and CartPage.razor components

![image](https://github.com/user-attachments/assets/af1528df-fda1-4beb-b707-372bb1d133ff)

## 17. We Add Nuget packages (WebApp project)

![image](https://github.com/user-attachments/assets/962036e1-9ea9-4828-99b2-2bd90ea1c374)

**Grpc.AspNetCore.Server.ClientFactory**: is used at runtime for managing gRPC clients in an ASP.NET Core 

**Grpc.Tools**: is used during development to generate C# code from .proto files, which defines the gRPC service and message contracts

**Microsoft.AspNetCore.Authorization**: package is a core component of ASP.NET Core that provides support for authorization in web applications

**Authorization** is the process of determining whether a user or system has the necessary permissions to perform specific actions or access specific resources

## 18. We Modify the WebApp csproj file

**WebApp.csproj**

```csharp
<ItemGroup>
  <Protobuf Include="..\Basket.API\Proto\basket.proto" GrpcServices="Client" />
</ItemGroup>
```

## 19. We Add Basket Services in the WebApp project

![image](https://github.com/user-attachments/assets/dedf6829-e743-4cf0-aa7f-632c3d3812e5)

This C# code defines a class called **BasketCheckoutInfo** that serves as a data transfer object (**DTO**) for handling basket **checkout information** in an e-commerce application

It likely captures the necessary details for processing a purchase and submitting a checkout request

**BasketCheckoutInfo.cs**

```csharp
using System.ComponentModel.DataAnnotations;

namespace eShop.WebApp.Services;

public class BasketCheckoutInfo
{
    [Required]
    public string? Street { get; set; }
    [Required]
    public string? City { get; set; }
    [Required]
    public string? State { get; set; }
    [Required]
    public string? Country { get; set; }
    [Required]
    public string? ZipCode { get; set; }
    public string? CardNumber { get; set; }
    public string? CardHolderName { get; set; }
    public string? CardSecurityNumber { get; set; }
    public DateTime? CardExpiration { get; set; }
    public int CardTypeId { get; set; }
    public string? Buyer { get; set; }
    public Guid RequestId { get; set; }
}
```

**BasketItem.cs**

This C# code defines a class called BasketItem that represents an **item in a shopping basket** in an e-commerce application

```csharp
namespace eShop.WebApp.Services;

public class BasketItem
{
    public required string Id { get; set; }
    public int ProductId { get; set; }
    public required string ProductName { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal OldUnitPrice { get; set; }
    public int Quantity { get; set; }
}
```

**IBasketState.cs**

```csharp
using eShop.WebAppComponents.Catalog;

namespace eShop.WebApp.Services
{
    public interface IBasketState
    {
        public Task<IReadOnlyCollection<BasketItem>> GetBasketItemsAsync();
        public Task AddAsync(CatalogItem item);
    }
}

```

The **BasketState** class provides a comprehensive solution for managing the shopping basket in an e-commerce app

It centralizes basket-related logic, integrates with other services, and maintains a reactive design to handle UI updates and state changes

The **BasketState** class:

**Manages Basket State**: Retrieves, updates, and deletes basket items. Provides caching to reduce redundant operations

**Handles Authentication**: Ensures basket operations are tied to the authenticated user

**Supports Change Notifications**: Notifies subscribers (e.g., UI components) of state changes

**Enables Checkout**: Manages basket checkout and prepares data for order creation

**BasketState.cs**

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Components;
using Microsoft.AspNetCore.Components.Authorization;
using eShop.WebAppComponents.Catalog;
using eShop.WebAppComponents.Services;

namespace eShop.WebApp.Services;

public class BasketState(
    BasketService basketService,
    CatalogService catalogService,
    //OrderingService orderingService,
    AuthenticationStateProvider authenticationStateProvider) : IBasketState
{
    private Task<IReadOnlyCollection<BasketItem>>? _cachedBasket;
    private HashSet<BasketStateChangedSubscription> _changeSubscriptions = new();

    public Task DeleteBasketAsync()
        => basketService.DeleteBasketAsync();

    public async Task<IReadOnlyCollection<BasketItem>> GetBasketItemsAsync()
        => (await GetUserAsync()).Identity?.IsAuthenticated == true
        ? await FetchBasketItemsAsync()
        : [];

    public IDisposable NotifyOnChange(EventCallback callback)
    {
        var subscription = new BasketStateChangedSubscription(this, callback);
        _changeSubscriptions.Add(subscription);
        return subscription;
    }

    public async Task AddAsync(CatalogItem item)
    {
        var items = (await FetchBasketItemsAsync()).Select(i => new BasketQuantity(i.ProductId, i.Quantity)).ToList();
        bool found = false;
        for (var i = 0; i < items.Count; i++)
        {
            var existing = items[i];
            if (existing.ProductId == item.Id)
            {
                items[i] = existing with { Quantity = existing.Quantity + 1 };
                found = true;
                break;
            }
        }

        if (!found)
        {
            items.Add(new BasketQuantity(item.Id, 1));
        }

        _cachedBasket = null;
        await basketService.UpdateBasketAsync(items);
        await NotifyChangeSubscribersAsync();
    }

    public async Task SetQuantityAsync(int productId, int quantity)
    {
        var existingItems = (await FetchBasketItemsAsync()).ToList();
        if (existingItems.FirstOrDefault(row => row.ProductId == productId) is { } row)
        {
            if (quantity > 0)
            {
                row.Quantity = quantity;
            }
            else
            {
                existingItems.Remove(row);
            }

            _cachedBasket = null;
            await basketService.UpdateBasketAsync(existingItems.Select(i => new BasketQuantity(i.ProductId, i.Quantity)).ToList());
            await NotifyChangeSubscribersAsync();
        }
    }

    public async Task CheckoutAsync(BasketCheckoutInfo checkoutInfo)
    {
        if (checkoutInfo.RequestId == default)
        {
            checkoutInfo.RequestId = Guid.NewGuid();
        }

        //var buyerId = await authenticationStateProvider.GetBuyerIdAsync() ?? throw new InvalidOperationException("User does not have a buyer ID");
        //var userName = await authenticationStateProvider.GetUserNameAsync() ?? throw new InvalidOperationException("User does not have a user name");

        // Get details for the items in the basket
        var orderItems = await FetchBasketItemsAsync();

        // Call into Ordering.API to create the order using those details
        //var request = new CreateOrderRequest(
        //    UserId: buyerId,
        //    UserName: userName,
        //    City: checkoutInfo.City!,
        //    Street: checkoutInfo.Street!,
        //    State: checkoutInfo.State!,
        //    Country: checkoutInfo.Country!,
        //    ZipCode: checkoutInfo.ZipCode!,
        //    CardNumber: "1111222233334444",
        //    CardHolderName: "TESTUSER",
        //    CardExpiration: DateTime.UtcNow.AddYears(1),
        //    CardSecurityNumber: "111",
        //    CardTypeId: checkoutInfo.CardTypeId,
        //    Buyer: buyerId,
        //    Items: [.. orderItems]);
        //await orderingService.CreateOrder(request, checkoutInfo.RequestId);
        await DeleteBasketAsync();
    }

    private Task NotifyChangeSubscribersAsync()
        => Task.WhenAll(_changeSubscriptions.Select(s => s.NotifyAsync()));

    private async Task<ClaimsPrincipal> GetUserAsync()
        => (await authenticationStateProvider.GetAuthenticationStateAsync()).User;

    private Task<IReadOnlyCollection<BasketItem>> FetchBasketItemsAsync()
    {
        return _cachedBasket ??= FetchCoreAsync();

        async Task<IReadOnlyCollection<BasketItem>> FetchCoreAsync()
        {
            var quantities = await basketService.GetBasketAsync();
            if (quantities.Count == 0)
            {
                return [];
            }

            // Get details for the items in the basket
            var basketItems = new List<BasketItem>();
            var productIds = quantities.Select(row => row.ProductId);
            var catalogItems = (await catalogService.GetCatalogItems(productIds)).ToDictionary(k => k.Id, v => v);
            foreach (var item in quantities)
            {
                var catalogItem = catalogItems[item.ProductId];
                var orderItem = new BasketItem
                {
                    Id = Guid.NewGuid().ToString(), // TODO: this value is meaningless, use ProductId instead.
                    ProductId = catalogItem.Id,
                    ProductName = catalogItem.Name,
                    UnitPrice = catalogItem.Price,
                    Quantity = item.Quantity,
                };
                basketItems.Add(orderItem);
            }

            return basketItems;
        }
    }

    private class BasketStateChangedSubscription(BasketState Owner, EventCallback Callback) : IDisposable
    {
        public Task NotifyAsync() => Callback.InvokeAsync();
        public void Dispose() => Owner._changeSubscriptions.Remove(this);
    }
}

public record CreateOrderRequest(
    string UserId,
    string UserName,
    string City,
    string Street,
    string State,
    string Country,
    string ZipCode,
    string CardNumber,
    string CardHolderName,
    DateTime CardExpiration,
    string CardSecurityNumber,
    int CardTypeId,
    string Buyer,
    List<BasketItem> Items);
```

**BasketService.cs**

The **BasketService** class is a service layer that interacts with a **gRPC-based basket API** to manage the shopping basket in an e-commerce application

It communicates with the basket API using the gRPC client (GrpcBasketClient) and provides methods for basket operations

**Integration with gRPC**: The BasketService uses a gRPC client (GrpcBasketClient) to communicate with the basket API

The gRPC methods (GetBasketAsync, DeleteBasketAsync, and UpdateBasketAsync) enable fetching, deleting, and updating basket items

**Abstraction**: The service abstracts the underlying gRPC communication, exposing simpler, strongly-typed methods for use within the application

Example: Instead of working directly with CustomerBasketResponse, the method returns a collection of BasketQuantity

**Mapping Data**: Converts gRPC response objects (like CustomerBasketResponse) into application-friendly data structures (BasketQuantity) for easier use

```csharp
using eShop.Basket.API.Grpc;
using GrpcBasketItem = eShop.Basket.API.Grpc.BasketItem;
using GrpcBasketClient = eShop.Basket.API.Grpc.Basket.BasketClient;

namespace eShop.WebApp.Services;

public class BasketService(GrpcBasketClient basketClient)
{
    public async Task<IReadOnlyCollection<BasketQuantity>> GetBasketAsync()
    {
        var result = await basketClient.GetBasketAsync(new ());
        return MapToBasket(result);
    }

    public async Task DeleteBasketAsync()
    {
        await basketClient.DeleteBasketAsync(new DeleteBasketRequest());
    }

    public async Task UpdateBasketAsync(IReadOnlyCollection<BasketQuantity> basket)
    {
        var updatePayload = new UpdateBasketRequest();

        foreach (var item in basket)
        {
            var updateItem = new GrpcBasketItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
            };
            updatePayload.Items.Add(updateItem);
        }

        await basketClient.UpdateBasketAsync(updatePayload);
    }

    private static List<BasketQuantity> MapToBasket(CustomerBasketResponse response)
    {
        var result = new List<BasketQuantity>();
        foreach (var item in response.Items)
        {
            result.Add(new BasketQuantity(item.ProductId, item.Quantity));
        }

        return result;
    }
}

public record BasketQuantity(int ProductId, int Quantity);
```

## 20. We Add the CartMenu razor component in the HeaderBar razor component (WebApp project)

We include the **CartMenu** in the **HeaderBar.razor**

![image](https://github.com/user-attachments/assets/db5958e4-042b-4f2b-bafe-ac8e4e0ceeb8)

**HeaderBar.razor**

```razor
@using Microsoft.AspNetCore.Components.Endpoints
@using Microsoft.AspNetCore.Components.Sections;

<div class="eshop-header @(IsCatalog? "home" : "")">
    <div class="eshop-header-hero">
        @{
            var headerImage = IsCatalog ? "images/header-home.webp" : "images/header.webp";
        }
        <img role="presentation" src="@headerImage" />
    </div>
    <div class="eshop-header-container">
        <nav class="eshop-header-navbar">
            <a class="logo logo-header" href="">
                <img alt="AdventureWorks" src="images/logo-header.svg" class="logo logo-header" />
            </a>
            
            <UserMenu />
            <CartMenu />
        </nav>
        <div class="eshop-header-intro">
            <h1><SectionOutlet SectionName="page-header-title" /></h1>
            <p><SectionOutlet SectionName="page-header-subtitle" /></p>
        </div>
    </div>
</div>

@code {
    [CascadingParameter]
    public HttpContext? HttpContext { get; set; }

    // We can use Endpoint Metadata to determine the page currently being visited
    private Type? PageComponentType => HttpContext?.GetEndpoint()?.Metadata.OfType<ComponentTypeMetadata>().FirstOrDefault()?.Type;
    private bool IsCatalog => PageComponentType == typeof(Pages.Catalog.Catalog);
}
```

## 21. We define the CartMenu.razor file

![image](https://github.com/user-attachments/assets/067fb8cc-d730-4b32-b88d-0ce41ad0de1f)

**CartMenu.razor**

```razor
@using System.Net
@attribute [StreamRendering]
@inject BasketState Basket
@inject LogOutService LogOutService
@inject NavigationManager NavigationManager
@implements IDisposable

<a aria-label="cart" href="cart">
    <img role="presentation" src="icons/cart.svg" />
    @if (basketItems?.Count > 0)
    {
        <span class="cart-badge">@TotalQuantity</span>
    }
</a>

@code {
    IDisposable? basketStateSubscription;
    private IReadOnlyCollection<BasketItem>? basketItems;

    [CascadingParameter]
    public HttpContext? HttpContext { get; set; }

    private int? TotalQuantity => basketItems?.Sum(i => i.Quantity);

    protected override async Task OnInitializedAsync()
    {
        // The basket contents may change during the lifetime of this component (e.g., when an item is
        // added during the current request). If this EventCallback is invoked, it will cause this
        // component to re-render with the updated data.
        basketStateSubscription = Basket.NotifyOnChange(
            EventCallback.Factory.Create(this, UpdateBasketItemsAsync));

        try
        {
            await UpdateBasketItemsAsync();
        }
        catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
        {
            await LogOutService.LogOutAsync(HttpContext!);
        }
    }

    public void Dispose()
    {
        basketStateSubscription?.Dispose();
    }

    private async Task UpdateBasketItemsAsync()
    {
        basketItems = await Basket.GetBasketItemsAsync();
    }
}
```

## 22. We Add the CartPage razor component (WebApp project)

This Razor Component represents a **product details page** for an e-commerce application built with Blazor

The page displays **detailed information about a product**, allows users to add it to their shopping cart, and adjusts its behavior based on the user's authentication status

This **product details page** is a feature-rich component that:

**Displays Product Information**: Dynamically loads and shows product details like name, price, description, and brand

**Handles User Interaction**: Allows authenticated users to add products to the cart

Redirects unauthenticated users to the login page

**Manages State**: Tracks the product's presence in the cart and updates it dynamically

**Error Handling**: Gracefully handles missing products by showing a "Not Found" message

This design ensures a user-friendly experience for browsing and interacting with products in an e-commerce application

**ItemPage.razor**

```razor
@page "/item/{itemId:int}"
@using System.Net
@inject CatalogService CatalogService
@inject BasketState BasketState
@inject NavigationManager Nav
@inject IProductImageUrlProvider ProductImages

@if (item is not null)
{
    <PageTitle>@item.Name | AdventureWorks</PageTitle>
    <SectionContent SectionName="page-header-title">@item.Name</SectionContent>
    <SectionContent SectionName="page-header-subtitle">@item.CatalogBrand?.Brand</SectionContent>

    <div class="item-details">
        <img alt="@item.Name" src="@ProductImages.GetProductImageUrl(item)" />
        <div class="description">
            <p>@item.Description</p>
            <p>
                Brand: <strong>@item.CatalogBrand?.Brand</strong>
            </p>
            <form class="add-to-cart" method="post" @formname="add-to-cart" @onsubmit="@AddToCartAsync" data-enhance="@isLoggedIn">
                <AntiforgeryToken />
                <span class="price">$@item.Price.ToString("0.00")</span>

                @if (isLoggedIn)
                {
                    <button type="submit" title="Add to basket">
                        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" xmlns="http://www.w3.org/2000/svg">
                            <path id="Vector" d="M6 2L3 6V20C3 20.5304 3.21071 21.0391 3.58579 21.4142C3.96086 21.7893 4.46957 22 5 22H19C19.5304 22 20.0391 21.7893 20.4142 21.4142C20.7893 21.0391 21 20.5304 21 20V6L18 2H6Z" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" />
                            <path id="Vector_2" d="M3 6H21" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" />
                            <path id="Vector_3" d="M16 10C16 11.0609 15.5786 12.0783 14.8284 12.8284C14.0783 13.5786 13.0609 14 12 14C10.9391 14 9.92172 13.5786 9.17157 12.8284C8.42143 12.0783 8 11.0609 8 10" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" />
                        </svg>
                        Add to shopping bag
                    </button>
                }
                else
                {
                    <button type="submit" title="Log in to purchase">
                        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" xmlns="http://www.w3.org/2000/svg">
                            <path d="M20 21V19C20 17.9391 19.5786 16.9217 18.8284 16.1716C18.0783 15.4214 17.0609 15 16 15H8C6.93913 15 5.92172 15.4214 5.17157 16.1716C4.42143 16.9217 4 17.9391 4 19V21" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" />
                            <path d="M12 11C14.2091 11 16 9.20914 16 7C16 4.79086 14.2091 3 12 3C9.79086 3 8 4.79086 8 7C8 9.20914 9.79086 11 12 11Z" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" />
                        </svg>
                        Log in to purchase
                    </button>
                }
            </form>

            @if (numInCart > 0)
            {
                <p><strong>@numInCart</strong> in <a href="cart">shopping bag</a></p>
            }
        </div>
    </div>
}
else if (notFound)
{
    <SectionContent SectionName="page-header-title">Not found</SectionContent>
    <div class="item-details">
        <p>Sorry, we couldn't find any such product.</p>
    </div>
}

@code {
    private CatalogItem? item;
    private int numInCart;
    private bool isLoggedIn;
    private bool notFound;

    [Parameter]
    public int ItemId { get; set; }

    [CascadingParameter]
    public HttpContext? HttpContext { get; set; }

    protected override async Task OnInitializedAsync()
    {
        try
        {
            isLoggedIn = HttpContext?.User.Identity?.IsAuthenticated == true;
            item = await CatalogService.GetCatalogItem(ItemId);
            await UpdateNumInCartAsync();
        }
        catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            HttpContext!.Response.StatusCode = 404;
            notFound = true;
        }
    }

    private async Task AddToCartAsync()
    {
        if (!isLoggedIn)
        {
            Nav.NavigateTo(Pages.User.LogIn.Url(Nav));
            return;
        }

        if (item is not null)
        {
            await BasketState.AddAsync(item);
            await UpdateNumInCartAsync();
        }
    }

    private async Task UpdateNumInCartAsync()
    {
        var items = await BasketState.GetBasketItemsAsync();
        numInCart = items.FirstOrDefault(row => row.ProductId == ItemId)?.Quantity ?? 0;
    }
}
```

## 23. We Modify the Extensions Middleware (WebApp project)

We register the **Basket services**

**Extensions.cs**

```csharp
public static void AddApplicationServices(this IHostApplicationBuilder builder)
{
  builder.AddAuthenticationServices();

  builder.Services.AddHttpForwarderWithServiceDiscovery();

  // Application services
  builder.Services.AddScoped<BasketState>();
  builder.Services.AddScoped<LogOutService>();
  builder.Services.AddSingleton<BasketService>();

  builder.Services.AddSingleton<IProductImageUrlProvider, ProductImageUrlProvider>();

  builder.Services.AddGrpcClient<Basket.BasketClient>(o => o.Address = new("http://localhost:5134"))
      .AddAuthToken();

  builder.Services.AddHttpClient<CatalogService>(o => o.BaseAddress = new("http://localhost:5301"))
      .AddApiVersion(1.0)
      .AddAuthToken();
}
```

## 24. We Add the Grpc Client Instrumentation (eShop.ServiceDefaults project)

We add the Nuget package and then we add the code

![image](https://github.com/user-attachments/assets/1469b7e5-82b2-49bd-96a0-e3b22c7d34e2)

## 25. We Run the Application and verify the results

First of all please confirm the Nuget Packages versions are equal, in this case 9.0.0

![image](https://github.com/user-attachments/assets/1866ebb2-b7af-47e8-9a73-11cb11fbdd46)

Then we run the application and we navigate to the Dashboard URL

https://localhost:17090/

![image](https://github.com/user-attachments/assets/215d265b-6045-4fd7-ba5d-3ad31308aa3e)

After running the application we can see the new **Cart icon**

![image](https://github.com/user-attachments/assets/37ccad8c-b690-4221-b561-dce22823a344)

We login in the application and we navigate to the **Cart Page**

![image](https://github.com/user-attachments/assets/39d79fb3-c6e6-49cd-8c52-81591d62b1e9)

We can navigate to an Item Page and then we can press in the **Add to Cart** button

![image](https://github.com/user-attachments/assets/86fcd3e9-e30e-48a3-ad3a-4fc6a3162898)

We confirm the new item was added in the Cart

![image](https://github.com/user-attachments/assets/0121bda6-a808-4be1-bbdf-2ff82dd8aea1)

![image](https://github.com/user-attachments/assets/337733b8-fbb9-48eb-8016-e5b484f6580a)


