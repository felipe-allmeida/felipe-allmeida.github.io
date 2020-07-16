# Software Testing in a .NET Core 3.1 web application

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

## Functional or end-to-end (e2e) tests

This guys are the real deal. They are used to simulate a **full user-level experience**. Things like:
* **Click on subscribe** - Should add you're e-mail to the subscribers list and whenever there is a new notification update all subscribed users.
* **Click on create** - Should subimit a form containing the model and save it on the database.
* **LogIn** - Should require e-mail and a password and when logged allow the user to access the web site.

So, for this I'll we will create a similar code from the one we used on the integration tests, check out:
```C#
public static class Server
{
    public static TestServer CreateServer()
    {
        var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
        
        var host = new WebHostBuilder()
            .UseEnvironment(environment)
            .ConfigureAppConfiguration((hostContext, configApp) =>
            {
                var path = GetBasePathFromAssembly(typeof(Startup));
                configApp.SetBasePath(path);
                configApp.AddJsonFile("appsettings.json", optional: true);
                configApp.AddJsonFile($"appsettings.{hostContext.HostingEnvironment.EnvironmentName}.json", optional: true);
            })
            .UseStartup<Startup>();
                    
        return new TestServer(host);
    }
}

private static string GetBasePathFromAssembly(Type type)
{
    string codeBase = Assembly.GetAssembly(type).CodeBase;
    UriBuilder uri = new UriBuilder(codeBase);
    string path = Uri.UnescapeDataString(uri.Path);
    return Path.GetDirectoryName(path);
}
```
The above example it's creating a host, using the `Startup` file and configurations of our the api. Basically it is simulating our real environment. The test code itself is similar to the integration in our case because we have only a web api, but if we had several other api's to consume, we would just add the needed communication to do the test

## Health Checks

This tests, as the name suggests, has the objective of validating if our project is alright. In our case, our API consumes and writes in a database, so in order for or API works healthy it **must** be connected to our database.

We can perform a simple check by just trying the command `SELECT 1` on our database. If it succeeds it means we are connected, so we return **healthy** as a response, otherwise **unhealthy**.
Startup.cs
```C#
public class Startup
{    
    public void ConfigureServices(IServiceCollection services)
    {
        /*
        * Your code....
        */    
        services.AddHealthChecks().AddCheck("my-database-name", new DatabaseHealthCheck("my-database-connection-string"));
    }
    
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        /*
        * Your code....
        */         
        app.UseHealthChecks("/health", new HealthCheckOptions
        {
            AllowCachingResponses = false,
            ResponseWriter = async (c, r) =>
            {
                c.Response.ContentType = "application/json";

                var results = r.Entries.Select(pair =>
                {
                    return KeyValuePair.Create(pair.Key, new ResponseResults
                    {
                        Status = pair.Value.Status.ToString(),
                        Description = pair.Value.Description,
                        Duration = pair.Value.Duration.TotalSeconds.ToString() + "s",
                        ExceptionMessage = pair.Value.Exception != null ? pair.Value.Exception.Message : "",
                        Data = pair.Value.Data
                    });
                }).ToDictionary(p => p.Key, p => p.Value);                
                
                var result = new HealthCheckResponse
                {
                    Status = r.Status.ToString(),
                    TotalDuration = r.TotalDuration.TotalSeconds.ToString() + "s",
                    Results = results
                };
                
                await c.Response.WriteAsync(JsonConvert.SerializeObject(result));
            };
        });
    }
}
```
DatabaseHealthCheck.cs
```C#
public class DatabaseHealthCheck : IHealthCheck
{
    private static readonly string DefaultTestQuery = "SELECT 1";
    private string ConnectionString { get; }
    private string TestQuery { get; }

    public DatabaseHealthCheck(string connectionString, string testQuery = null)
    {
        if (String.IsNullOrEmpty(connectionString)) throw new ArgumentNullException(nameof(connectionString));

        if (!String.IsNullOrEmpty(testQuery)) TestQuery = testQuery;
        else TestQuery = DefaultTestQuery;

        ConnectionString = connectionString;            
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var dataSource = Regex.Match(ConnectionString, @"Data Source=([A-Za-z0-9_.]+)", RegexOptions.IgnoreCase).Value;

        using (var connection = new MySqlConnection(ConnectionString))
        {
            try
            {
                await connection.OpenAsync(cancellationToken);

                var command = connection.CreateCommand();
                command.CommandText = TestQuery;

                await command.ExecuteNonQueryAsync(cancellationToken);

                return HealthCheckResult.Healthy(dataSource);
            }
            catch (Exception e)
            {
                return HealthCheckResult.Unhealthy(dataSource, e);
            }
        }
    }
}
```

## Testing in a continuous delivery pipeline (Azure Pipelines)

Let's use our tests in a continuous delivery pipeline to ensure quality to our software. Whenever our code is delivered to our master branch, we will automatically build, test and deploy it by using azure pipelines.

Here is a simple .yml file that execute the following pipeline steps on the master branch:
1. Run unit tests
2. Run integration tests
3. Deploy to azure
4. Run health checks
5. Run functional tests

```yml
trigger:
- master

pool:
  vmImage: ubuntu-16.04
 
name: PipelineName-${Date:yyyyMMdd}$(Rev:.r)

variables:
  ASPNETCORE_ENVIRONMENT: 'Production'
  buildConfiguration: 'Release'

steps:   
- task: DotNetCoreCLI@2
  displayName: 'unit tests'
  inputs:
    command: test
    projects: '**/*UnitTests/*.csproj'
    
- task: DotNetCoreCLI@2
  displayName: 'integration tests'
  inputs:
    command: test
    projects: '**/*IntegrationTests/*.csproj'

- task: DotNetCoreCLI@2
  displayName: 'dotnet publish'
  inputs:
    command: publish
    publishWebProjects: True
    arguments: '--configuration $(BuildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: True

- task: PublishBuildArtifacts@1
  inputs:
    pathtoPublish: '$(Build.ArtifactStagingDirectory)' 
    artifactName: 'myWebsiteName'

- task: AzureWebApp@1
  displayName: 'deploy artifacts'
  inputs:
    azureSubscription: ${{ parameters.azureSubscription }}
    appType: webApp
    appName: 'yourWebsiteName'
    package: $(System.ArtifactsDirectory)/**/*.zip 
    
- task: DotNetCoreCLI@2
      displayName: 'health checks'
      inputs:
        command: test
        projects: '**/*HealthChecks/*.csproj'
        
- task: DotNetCoreCLI@2
      displayName: 'functional tests'
      inputs:
        command: test
        projects: '**/*FunctionalTests/*.csproj'
```
This .yml is already working, you can just create a new pipeline on azure and add this code to run your pipeline!

## Conclusion

The tests on this article are fairly simple, this is how a test should be. In order to detect errors as soon as possible in our code, we also have fail fast, that means we need to have simple tests covering **our application** functionallity.

I advocate that every good developer test their code to ensure quality. There is no "I didn't have time to test", testing is proven to reduce problems, with no problems there is no extra hour and with no extra hour your company spend less money.

More tests = Less problems = Less money spent in the project
