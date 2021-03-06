# Using Rhetos 5 with ASP.NET

Sample app on how to add Rhetos to ASP.NET Web API project.

Complete source code for this example is available at: https://github.com/sstublic/Rhetos.Samples.AspNet

## Setting up

1. Create a new folder for your project

2. Run `dotnet new webapi`

3. Configure `.csproj`

Prevent Rhetos auto deploy:
Add `<RhetosDeploy>False</RhetosDeploy>` to `<PropertyGroup>` tag.

Add packages:
```
<ItemGroup>
  <PackageReference Include="Rhetos.Host" Version="5.0.0-dev*" />
  <PackageReference Include="Rhetos.Host.AspNet" Version="5.0.0-dev*" />
  <PackageReference Include="Rhetos.CommonConcepts" Version="5.0.0-dev*" />
  <PackageReference Include="Rhetos.MSBuild" Version="5.0.0-dev*" />
  <PackageReference Include="Microsoft.Extensions.Configuration" Version="5.0.0" />
</ItemGroup>
```

## Build your first Rhetos App

Add Rhetos DSL script named `DslScripts/Books.rhe` and add the following to it:

```
Module Bookstore
{
    Entity Book
    {
        ShortString Code { AutoCode; }
        ShortString Title;
        Integer NumberOfPages;

        ItemFilter CommonMisspelling 'book => book.Title.Contains("curiousity")';
        InvalidData CommonMisspelling 'It is not allowed to enter misspelled word "curiousity".';

        Logging;
    }
}

```

*This sample is in `Rhetos.*` namespace so we need to correct `Host` conflict in `Program.cs` by changing `Host.CreateDefaultBuilder(...` reads `Microsoft.Extensions.Hosting.Host.CreateDefaultBuilder(...`.*

Run `dotnet build` to verify that everything compiles. **Your DSL model from newly added script will be compiled and Rhetos classes are now available in your project.**

## Connecting to ASP.NET pipeline

To wire up Rhetos and ASP.NET dependency injection, modify `Startup.cs`, add a static method (this is a useful convention, explained later):

```
using Rhetos;
```

```
public static void ConfigureRhetosHostBuilder(IRhetosHostBuilder rhetosHostBuilder, Microsoft.Extensions.Configuration.IConfiguration configuration)
{
    rhetosHostBuilder
        .ConfigureRhetosHostDefaults()
        .ConfigureConfiguration(cfg => cfg.MapNetCoreConfiguration(configuration));
}
```

And register Rhetos in `ConfigureServices` method:

```
services.AddRhetos(rhetosHostBuilder => ConfigureRhetosHostBuilder(rhetosHostBuilder, Configuration))
    .UseAspNetCoreIdentityUser();
```

Rhetos needs database to work with, create it and configure connection string in `appsettings.json` file:

```
  "ConnectionStrings": {
    "RhetosConnectionString": "<YOURDBCONNECTIONSTRING>"
  }
```

## Applying Rhetos model to database

To apply model to database we need to use `rhetos.exe` CLI tool. CLI tools need to be able to discover host application configuration and setup. We provide that via static method in `Program.cs`.
Add the following to `Program.cs`

```
using Microsoft.Extensions.DependencyInjection;
```

```
public static IRhetosHostBuilder CreateRhetosHostBuilder()
{
    var host = CreateHostBuilder(null).Build();
    var configuration = host.Services.GetRequiredService<IConfiguration>();

    var rhetosHostBuilder = new RhetosHostBuilder();
    Startup.ConfigureRhetosHostBuilder(rhetosHostBuilder, configuration);

    return rhetosHostBuilder;
}
```

`rhetos.exe` will discover and use this method by convention to construct a Rhetos host.
We are reusing `Startup.ConfigureRhetosHostBuilder` so that our Rhetos is configured exactly the same as in runtime.

Run `dotnet build`

Run `./rhetos.exe dbupdate Rhetos.Samples.AspNet.dll` in the binary output folder. This runs database update operation in the context of specified host DLL (in our case, our sample application).

## Inject Rhetos components into ASP.NET controllers

Add a new controller `MyRhetosController.cs`.

```
using Microsoft.AspNetCore.Mvc;
using Rhetos.Host.AspNet;
using Rhetos.Processing;

[Route("Rhetos/[action]")]
public class MyRhetosController : ControllerBase
{
    private readonly IRhetosComponent<IProcessingEngine> rhetosProcessingEngine;

    public MyRhetosController(IRhetosComponent<IProcessingEngine> rhetosProcessingEngine)
    {
        this.rhetosProcessingEngine = rhetosProcessingEngine;
    }

    [HttpGet]
    public string HelloRhetos()
    {
        return rhetosProcessingEngine.Value.ToString();
    }
}
```

Run `dotnet run` and browse to `http://localhost:5000/Rhetos/HelloRhetos`. You should see the name of the `ProcessingEngine` type meaning we have successfully resolved it from Rhetos:
`Rhetos.Processing.ProcessingEngine`.

## Executing Rhetos commands

Add a method to `MyRhetosController.cs` to read our Books entity.

```
using Rhetos.Processing.DefaultCommands;
using System.Collections.Generic;
using System.Linq;
```

```
[HttpGet]
public string ReadBooks()
{
    var readCommandInfo = new ReadCommandInfo() { DataSource = "Bookstore.Book", ReadTotalCount = true };

    var processingResult = rhetosProcessingEngine.Value.Execute(new List<ICommandInfo>() {readCommandInfo});
    var result = (ReadCommandResult) processingResult.CommandResults.Single().Data.Value;
    return result.TotalCount.ToString();
}
```

By default, Rhetos permissions will not allow anonymous users to read any data. Enable anonymous access by modifying `appsettings.json`:

```
"Rhetos": {
  "AppSecurity": {
    "AllClaimsForAnonymous": true
  }
}
```

Run the example and navigate to `http://localhost:5000/Rhetos/ReadBooks`. You should receive a response value `0` indicating there are 0 entries in our book repository.

# Additional integration/extension options

## Adding ASP.NET authentication and connecting it to Rhetos

**In this example we will use the simplest possible authentication method, although ANY authentication method supported by ASP.NET may be used. For example [Configure Windows Authentication](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/windowsauth?view=aspnetcore-5.0&tabs=visual-studio)**

Add authentication to ASPNET application. Modify `Services.cs`:

```
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Http;
```

Add to `ConfigureServices`:

```
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(o => o.Events.OnRedirectToLogin = context =>
    {
        context.Response.StatusCode = StatusCodes.Status401Unauthorized;
        return Task.CompletedTask;
    });
```

And in `Configure` method after `UseRouting()` add:
```
app.UseAuthentication();
```

Modify `MyRhetosController.cs`
```
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using System.Threading.Tasks;
using System.Security.Claims;
```

and a new method to allow us to sign-in:

```
[HttpGet]
public async Task Login()
{
    var claimsIdentity = new ClaimsIdentity(new[] { new Claim(ClaimTypes.Name, "SampleUser") }, CookieAuthenticationDefaults.AuthenticationScheme);

    await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme,
        new ClaimsPrincipal(claimsIdentity),
        new AuthenticationProperties() { IsPersistent = true });
}
```

This is simple stub code to sign-in `SampleUser` so we have a valid user to work with.

In `appsettings.json` set `AllClaimsForAnonymous` to `false`. This disables anonymous workaround we have been using so far.

If you run the app now and navigate to `http://localhost:5000/Rhetos/Login` and then to `http://localhost:5000/Rhetos/ReadBooks`, you will receive an error:
`UserException: Your account 'SampleUser' is not registered in the system. Please contact the system administrator.`

Since 'SampleUser' doesn't exist in Rhetos we will use a simple configuration feature to treat him as admin.

Add to `appsettings.json`:
```
"Rhetos": {
  "AppSecurity": {
    "AllClaimsForUsers": "SampleUser@<YOURMACHINENAME>"
  }
}
```

`http://localhost:5000/Rhetos/ReadBooks` should now correctly return `0` as we haven't added any `Book` entities.

You can write additional controllers/actions and invoke Rhetos commands now.

## Adding Rhetos.RestGenerator

Rhetos.RestGenerator package automatically maps all Rhetos datastructures to REST endpoints.

Add package to `.csproj` file:

```
<PackageReference Include="Rhetos.RestGenerator" Version="5.0.0-dev*" />
```

Modify lines which add Rhetos in `Startup.cs`, method `ConfigureServices` to read:

```
services.AddRhetos(rhetosHostBuilder => ConfigureRhetosHostBuilder(rhetosHostBuilder, configuration))
    .UseAspNetCoreIdentityUser()
    .AddRestApi(o => o.BaseRoute = "rest");
```

Run `dotnet run`. REST API is now available. Navigate to `http://localhost:5000/rest/Bookstore/Book` to issue a GET and retrieve all Book entity records in the database.

For more info on usage and serialization configuration see [Rhetos.RestGenerator](https://github.com/Rhetos/RestGenerator)

### View endpoints in Swagger

Since Swagger is already added to webapi project template, we can generate Open API specification for mapped Rhetos endpoints.

Modify lines which add Rhetos in `Startup.cs`, method `ConfigureServices` to read:

```
services.AddRhetos(rhetosHostBuilder => ConfigureRhetosHostBuilder(rhetosHostBuilder, Configuration))
    .UseAspNetCoreIdentityUser()
    .AddRestApi(o => 
    {
        o.BaseRoute = "rest";
        o.GroupNameMapper = (conceptInfo, name) => "v1";
    });
```

This addition maps all generated Rhetos API controllers to an existing Swagger document named 'v1'.

Run `dotnet run Environment=Development` and navigate to `http://localhost:5000/swagger/index.html`. You should see entire Rhetos REST API in interactive UI.
