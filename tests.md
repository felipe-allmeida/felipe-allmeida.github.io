# Continuous testing on a .NET Core 3.1 application - Part 1

What's up everyone?

This is the first part of this article series. In it we will learn what they are and create unit, integration and functional tests as well as put them in our deployment pipeline on Azure.

In this article I will cover:
* Unit Tests
* Integration Tests

## Requirements

.NET Core 3.1: https://dotnet.microsoft.com/download/dotnet-core/3.1
Visual Studio: https://visualstudio.microsoft.com/pt-br/downloads/

## What are Unit Tests?

Unit Testing is the practice of testing small pieces of code, typycally individual functions, alone and isolated. If you have to access some external service, it's not a unit test.

Here we have a simple one, using [Xunit](https://www.nuget.org/packages/xunit/) that checks if a product is on stock:
```C#
[Fact]
public void DebitStock_ShouldDebitValueFromStock()
{
    var product = new Product() { StockQuantity = 10 };
    
    product.DebitStock(1);
    
    Assert.Equal(9, product.StockQuantity);
}
```
Fairly simple right? And yet we're testing a important funcionallity from our application. I'll create another one, this time I expect an error.
```C#
[Fact]
public void DebitStock_WhenNoStockAvaiable_ShouldThrowException()
{
    var product = new Product() { StockQuantity = 10 };
    
    Assert.Throws<Exception>(() =>
    {
        product.DebitStock(11);    
    };
}
```
Just by reading this you can understend what are our business rules right? We have a functionallity to debit value from that product stock and in case the stock is not enough, it should throw an error.

Now imagine that for all our business rules we have small pieces of tests like the ones above covering our code, the smallest change in the rules would be detected by our tests. This way we're ensuring that our code is following the business needs.

## Integration Tests
Integration tests are very similar to unit tests, but with one big difference: while unit tests are isolated from external services, integration tests are not.

So, for this example I'll use a WebApi, we will have to build a testing environment in order to perform our tests:
```C#
public static class Server
{
    public static TestServer CreateServer()
    {
        var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
        
        var host = new WebHostBuilder()
            .UseEnvironment(environment)
            .ConfigureServices((hostingContext, services) =>
            {
                services.AddDbContext<AppDbContext>(opt => opt.UseInMemoryDatabase(databaseName: "InMemoryDb"));
                services.AddScoped<IProductService, ProductService>();
                
                services.AddMvc();
            })
            .Build();
            
        host.Configure((app) =>
        {
            app.UseMvc();
        });
            
        return new TestServer(host);
    }
}
```

Now that we have our test environment, let's perform a integration test:
```C#
[Fact]
public void Post_CreateProduct_ShouldCreateProduct()
{
    var server = Server.CreateServer();
    
    var client = server.CreateClient();
    
    var data = new ProductViewModel { Name = "MyFakeProduct", Price = 50, StockQuantity = 2 };
    var content = new StringContent(JsonConvert.SerializeObject(data), Encoding.UTF8, "application/json");
    
    var response = client.PostAsync("/api/product", content).Result;
    var viewModel = JsonConvert.DeserializeObject<ProductViewModel>(response.Content.ReadAsStringAsync().Result);
    
    Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    Assert.NotEqual(Guid.Empty, viewModel.Id);
}
```
As you can see, we create a whole host in order to load all dependencies needed in order to test the functionallity of registering a new product. 
Then, we start a post rest request for the webapi endpoint `/api/product` passing our new product as a parameter.

To validate the test, we check if the request response is a `201 Created` and if the productViewModel has returned a Id, that is, our product was created successfully.

## Conclusion

The tests on this article are fairly simple, this is how a test should be. In order to detect errors as soon as possible in our code, we also have fail fast, that means we need to have simple tests covering **our application** functionallity.

In the next part, we will set some functional tests and some health checks in our web api.

# References
* https://www.eduardopires.net.br/2017/07/ferramentas-para-escrever-testes-de-unidade-mais-eficientes/
* https://github.com/dotnet/roslyn
* https://docs.microsoft.com/pt-br/dotnet/core/testing/unit-testing-with-dotnet-test

