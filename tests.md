# Software Testing in a .NET Core 3.1 web application - Part 1

## Requirements

* [.NET Core 3.1](https://dotnet.microsoft.com/download/dotnet-core/3.1)
* [Visual Studio](https://visualstudio.microsoft.com/pt-br/downloads/)

## So, what is Software Testing?

> Is an organizational process within software development in which business-critical software is verified for correctness, quality and performance. Software testing is used to ensure that expected business systems and product features behave correctly as expected.
> Claire Maynard

Those tests can either be manual or an automated process:

* **Manual software testing**, led by a team or individual who will manually operate a software and ensure it behaves as expected by running some tests.
* **Automated software testing** is the automated version of the manual software testing.

## And about the benefits?

Testing will save tons of time **and money**. Testing will ensure that a feature is working as expected by the business, preventing users to find any bug while they use the software.

Also for developing new features, if we have the tests properly setup, the time to implement is reduced considerably because we already know what to output and that is already tested. And by running tests against an already delivered feature we ensure that our changes are not breaking anything that is already working.

## Unit Tests

Unit Testing is the practice of testing small pieces of code, typycally individual functions, alone and isolated. If you have to access some external service, it's not a unit test.

They have the goal to test small pieces of business requirements by passing an input and comparing a expected result with the code output.

Here's a simple one, using [Xunit](https://www.nuget.org/packages/xunit/) that checks if a product stock is being debited:
```C#
[Fact]
public void DebitStock_ShouldDebitValueFromStock()
{
    var product = new Product() { StockQuantity = 10 };
    
    product.DebitStock(1);
    
    Assert.Equal(9, product.StockQuantity);
}
```
Fairly simple right? And yet we're testing a **real funcionallity** from our application. I'll create another one, this time I'll test a error situation.
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
Integration tests are very similar to unit tests, but with one big difference: while unit tests are isolated from external services, integration tests are not. Integration tests 
confirm that two or more app components work together to produce an expected result.

So, let's go for a example where we only have one two components: a web api and a database.
In order to initialize both component's, we're going to create a test environment that will load only the necessary dependencies for the test:
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
With our dependencies loaded, now is the time to write a integration test:
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
