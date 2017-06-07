---
title: "ASPNET-Core-2.0-Stripping-Away-Cross-Cutting-Concerns"
excerpt: "Stuck with on Prem, or just aren't feeling the cloud? Here is how to get some coolness of your own"
category: "tim"
header:
  overlay_image: crosscut/scissors.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: crosscut/teaser.jpg
tags: [aspnet, webapi, aspnetcore, serilog, consul, dotnetcore store, IHostingStartup]
---

## The background

By now most of you have probably seen the cool concepts of ASPNETCORE 2.0 and Azure integration. That is you can build an application and then magically an admin or ops person can turn on cross cutting concerns that you never had to worry about at development time.
This is all well and good if you are hosting in the cloud but can we achieve something similar on-prem on hosted in our own container environment? Sure we can! So if your devops people decide they 
want to use a different logging library or add some other authentication method well they can now and the dev writing that business logic really doesn't need to do anything..

--Warning : This is using .net 2.0 Preview parts and these are likely to change, as we get closer to release I will write another article with any changes that have taken place--

[The code for this post](https://github.com/drawaes/condenser.apifirst/tree/DotNet2-Preview)

## Step one ... the basics

First up we create an ASPNETCORE 2.0 web project. To do this we are going to go self hosted from a plain vanilla Console App. But before we get to that you will need Visual Studio 2017 Preview 

[Visual Studio Preview](https://www.visualstudio.com/vs/preview/)

You can install which ever flavour you want here. This however isn't enough you also need to install DOTNET Core 2.0 Preview from the below link (the preview doesn't come bundled with Visual Studio 2017 Preview)

[.Net Core Preview](https://www.microsoft.com/net/core/preview#windowscmd)

We are not going to go throguh upgrading an existing project and all of the changes, Rick Strahl has an excellent blog post on [how to upgrade](https://weblog.west-wind.com/posts/2017/May/15/Upgrading-to-NET-Core-20-Preview).

If we look at a typical web application you might have something like this in your startup (although probably a lot more as this is just a sample).

Each of the numbers I have marked are cross cutting concerns, logging, registering the service with a service discovery platform and finally setting up Swagger/API definitions via Swashbuckle.

![Configure](https://cetus.io/images/crosscut/configure.png)

1,2 Setting up Serilog to output out logging, this can change between environments or depending on where/how and the format that ops want
3,4 Service Registration with Consul
5   Swashbuckle setup to provide Swagger API Documents

![Configure Services](https://cetus.io/images/crosscut/configureservices.png)

1,2 Adding Consul Service Registration and Shutdown
3   Swashbuckle Services

That all seems like a lot of boiler plate code for our devs to worry about and get right for every small service they want to push out. It also means that code is embeded in every application, so we could
make helper methods to ease the pain, make that into a package and then use that. It still leaves us with the problem of updating or changing that package and then updating out applications and redeploying.

## Enter our new best friend IHostingStartup

This new interface is pretty simple as follows

``` csharp
public class HostingBootStrap : IHostingStartup
{
    public void Configure(IWebHostBuilder builder)
    {
    }
}
```

It's a pretty minimal but very powerful interface, with ASPNET 2.0 having access to the IWebHostBuilder allows us to change almost everything, so first we can take out some of our boilerplate code

``` csharp
public void Configure(IWebHostBuilder builder)
{
    builder.ConfigureServices(services =>
    {
        services.AddConsulServices();
        services.AddConsulShutdown();
        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", 
                new Info 
                { 
                    Title = builder.GetSetting(WebHostDefaults.ApplicationKey), 
                    Version = "v1", 
                    License = new License() { Name = "MIT" } 
                });
            c.DescribeAllEnumsAsStrings();
        });

    });
}
```

So you can see here we have added our services inside the config services and generalized the setup by using the application name from the settings to provide the name to SwashBuckle. This ConfigureServices
is additive to any configure services that is in the startup class we provide so we don't have to worry about nuking the applications own service setup. This leaves our new Startup class looking like

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
}
```

Now we could go as far as adding the AddMvc to our new startup but this bootstrap could be used for "raw" services without Mvc so we will leave that up to the implementor. 

Next we come to the "Configure" method, this one is a little more tricky. Firstly we really want to move the logging up and out of the Configure. One reason for this is that we can't currently catch or
log anything until we get to the configure method which is well after service setup and a bunch of other bootstrapping tasks. To do this we will add it directly to the host as below

``` csharp
builder.ConfigureLogging(loggerFactory =>
{
    Log.Logger = new LoggerConfiguration()
        .Enrich.FromLogContext()
        .WriteTo.ColoredConsole(outputTemplate: "{Timestamp:yyyy-MMM-dd HH:mm:ss} [{Level}] [{scope}] {Message}{NewLine}{Exception}")
        .CreateLogger();
    loggerFactory.AddSerilog();
});
```

With that out of the way we are left with the single problem of our service registration and middleware registration. Now if you are thinking that you could just override the Configure but if you do that
only your Configure will run, it is last one wins. So what we need is a Startup Filter, the interface is below.

``` csharp
public class HostingStartupFilter : IStartupFilter
{
    public Action<IApplicationBuilder> Configure(Action<IApplicationBuilder> next)
    {
    }
}
```

And we hook it up by adding it to our servicecollection with

``` csharp
services.AddTransient<IStartupFilter, HostingStartupFilter>();
```

So now if we implement the startup filter interface we end up with something like

``` csharp
public class HostingStartupFilter : IStartupFilter
{
    private IServiceManager _serviceManager;
    private string _applicationName;

    public HostingStartupFilter(IServiceManager serviceManager, IHostingEnvironment hostingEnvironment)
    {
        _serviceManager = serviceManager;
        _applicationName = hostingEnvironment.ApplicationName;
    }

    public Action<IApplicationBuilder> Configure(Action<IApplicationBuilder> next)
    { 
        return app =>
        {
            _serviceManager
                .AddHttpHealthCheck("api/health", 60)
                .WithDeregisterIfCriticalAfterMinutes(1)
                .RegisterServiceAsync();

            app.UseConsulShutdown();

            next(app);

            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", _applicationName);
            });
        };
    }
}
```

Once again we have replaced the static name for the application name. One thing you can also see is that because the class is resolved from our DI container we can have services injected into the constructor. We are also running the Swagger setup after we call the next part of the pipeline of configure methods because we need it to run after any MVC setup.

So that is it our class that we want injected. If we had a single project as our web application this would be scanned and picked up automatically. However this isn't the case because we have our application
in a different class to our host. So we add the following attribute to the assembly (so add it outside the namespace).

``` csharp
[assembly:HostingStartup(typeof(Condenser.ApiFirst.DocumentStorage.Core.HostingBootStrap))]

namespace Condenser.ApiFirst.DocumentStorage.Core
{
```

The last part we have is setting up the hosting port/Uri. We can add this through the same IWebHostBuilder. This pretty much removes the need for anything but adding the startup class.
Now when we run the application we can sucessfully see our class and configuration being run.

## Ground work done, now for some magic

That is all well and good but all we have done is move a lot of our bootstrapping into a seperate class (well two if you include the setup filter) but we haven't moved it away from being tied into our application logic. It this point we will create a seperate dll for our project.

When we strip out the settings that we are now configuring from our bootstrapping class we are left with this in our "host"

``` csharp
static void Main(string[] args)
{
  var host = new WebHostBuilder()
    .UseStartup<Startup>()
    .Build();

  host.Run();
}
```

And our Startup class looks as follows

``` csharp
public class Startup
{
  public void Configure(IApplicationBuilder app) => app.UseMvc();
  public void ConfigureServices(IServiceCollection services) => services.AddMvc();
}
```

And finally our CSProj file looks something like

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.All" Version="2.0.0-preview1-final" />
    <ProjectReference Include="..\Condenser.ApiFirst.BootStrapper\Condenser.ApiFirst.BootStrapper.csproj" />
    <ProjectReference Include="..\Condenser.ApiFirst.SwaggerDoc.Core\Condenser.ApiFirst.SwaggerDoc.Core.csproj" />
  </ItemGroup>

</Project>
```

So there we have a reference to the new "mega package" that includes everything we may need for ASP.NET including Dapper that we are using amoung other things. The other reference is to our library for our application and then of course the boot strapper project. That is really it for moving out to a Dll. But it's still hard linked into the application and to get it to work we need the "assembly" attribute right there in our application, so we need to go to the next level.

## Time to Get Environmental

As we discussed before with the current setup we have to update the package when we want to change it, we also have to be "aware" that the package exists and add it to our project. So then how could we make this "Automagical"? The answer lies in some special environment variables, first the variable

```
ASPNETCORE_HOSTINGSTARTUPASSEMBLIES=Condenser.ApiFirst.BootStrapper
```

That's it, our dll is now loaded into the process. There is a problem however, if we remove the reference to the project for that assembly we run into an issue of dependencies. In DOTNET Core there is a magical file
called Your.Assembly.Name.Deps.json. This doesn't have to be present (default probing similar to traditional .Net will take place in that case) but if it is, it gives the runtime the dependency chain for your application. Below is a sample entry for the core condenserdotnet library

``` json
"CondenserDotNet.Core/2.6.5": {
  "dependencies": {
    "Microsoft.AspNetCore.Http": "2.0.0-preview1-final",
    "Microsoft.Extensions.Logging": "2.0.0-preview1-final",
    "Newtonsoft.Json": "10.0.1"
  },
  "runtime": {
    "lib/netstandard1.6/CondenserDotNet.Core.dll": {}
  }
},
```

So now the problem with our solution of injecting our assembly is the runtime doesn't know about our hosting assemblies dependencies. So step up the next in line for our exciting environment variables

```
DOTNET_ADDITIONAL_DEPS=C:\code\Condenser.ApiFirst\additionaldeps
```

This little gem will now tell the runtime to take a look in there to see if there are any dependency files that might match and merge them into our applications dependency tree. In order to match we need to have a 
specific directory structure under this folder. 

1. The Runtime type, in our case we will just put "shared" for now
2. The Framework type which we will have "Microsoft.NETCore.App"
3. The Framework version, in our case "2.0.0-preview1-002111-00"

So the final folder where we put our extra dependency file will be "C:\code\Condenser.ApiFirst\additionaldeps\shared\Microsoft.NETCore.App\2.0.0-preview1-002111-00". The file we will put in there will be the one straight from the build folder of our hosting injected application "Condenser.ApiFirst.BootStrapper.deps.json". 

Simple right? Wrong, if you have got to this point you might notice that it won't actually run.

![Failure to run](https://cetus.io/images/crosscut/missingdll.png)

This makes sense, we have told the runtime we have these dependencies, but where can it look for them? By default it will probe it's local directory for them and a couple of other places but it will come up empty handed.
So there are two main ways we can fix this and we will look at both.

## Putting a direct link in there

This is probably the most "hacky" way of adding dependencies, however it has a use case and that is our one now. We have a single Dll that we want to be able to change, we don't have it in Nuget inside our organisation
and we don't want to have our containers go and get it. Also during development we want the "latest" copy of it rather than having to do a number of steps. So if we go back and look in our "additional" deps.json file we can see this section at the top

``` json
".NETCoreApp,Version=v2.0": {
  "Condenser.ApiFirst.BootStrapper/1.0.0": {
  "dependencies": {
    "CondenserDotNet.Client": "2.6.5",
    "CondenserDotNet.Middleware": "2.6.5",
    "Microsoft.AspNetCore.All": "2.0.0-preview1-final",
    "Serilog.Extensions.Logging": "1.4.1-dev-10155",
    "Serilog.Sinks.ColoredConsole": "2.1.0-dev-00713",
    "Swashbuckle.AspNetCore": "1.0.0"
  },
  "runtime": {
    "Condenser.ApiFirst.BootStrapper.dll": {}
  }
},
```
The important bit is at the end. It is the dll that is the actual runtime for that dependency. So we can simply change it point directly to our debug folder (with some escaping because it is Json) 

"C:\\code\\Condenser.ApiFirst\\src\\Condenser.ApiFirst.BootStrapper\\bin\\Debug\netcoreapp2.0\\Condenser.ApiFirst.BootStrapper.dll"

This works but now when we run we hit more missing dependencies

![More Missing](https://cetus.io/images/crosscut/moremissingdlls.png)

So now the dependencies of our bootstrapper are missing, and these are packages with potentially many more missing/required dependencies and we don't want to hard code each dll... but don't worry there is a solution

## Enter the Store!

So if any of you remember the GAC, it's kinda like that, but kinda not (in a good way). ASPNET/DOTNET 2.0 actually comes preloaded with a store if you have the SDK, it contains that magical ASP.NET 2.0 "All" package and that is why if you are building locally and not "standalone" then your project size should be tiny. But let's pretend we don't want to install the sdk on every image but we have our standard set of libraries we want to include (or need to in the case of our injected dependencies) how do we make our own store?

Well first we start with an MSBuild style file that is basically a stripped down CSProj and put our package dependencies in it as normal

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="CondenserDotNet.Client" Version="2.6.5" />
    <PackageReference Include="CondenserDotNet.Middleware" Version="2.6.5" />
    <PackageReference Include="Microsoft.AspNetCore.All" Version="2.0.0-preview1-final" />
    <PackageReference Include="Serilog.Extensions.Logging" Version="1.4.1-dev-10155" />
    <PackageReference Include="Serilog.Sinks.ColoredConsole" Version="2.1.0-dev-00713" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="1.0.0" />
  </ItemGroup>
</Project>
```

Then we run the command

dotnet store --manifest Condenser.ApiFirst.Store.csproj --runtime win10-x64 --framework netcoreapp2.0 --output store

Breaking this down we are saying we want to import the packages from the csproj file. It will chain down and get the whole tree required. We want to build for win10-x64 (later we will look at doing this for
docker but for now you just want to do it for your local machine) and finally we want to compile for netcoreapp2.0 and output the results to a folder called store. If you leave off the output it will actually import it directly into the store that you will have from the SDK. We don't want to do this for now because we want to take a look at the output.

The first thing you will notice it that it will try to precompile the files as much as possible for your platform (you can surpress this using skip optimization). This should improve load times. If we look
in the output folder we will see a file called "artifact.xml". This file is important as it has a list of all of the packages/dependencies the store will now have in it. Now we will run the same command but witout the output folder to actually import the dependecies into the dotnet store and try to rerun our application that failed before and we see

![WithEnvironmentVars](https://cetus.io/images/crosscut/productionenvironmentvars.png)

And without the environment variables 

![WithoutEnvironmentVars](https://cetus.io/images/crosscut/noenvironmentvars.png)

So what did we achieve? 

1. Our application has no knowledge of the logging, port allocation or service discovery
2. We can have it on or off in different environments
3. We could change the implementation completely without touching the application or rebuilding
4. We have learnt to bring in libraries to the store
5. We have seen the manifest output ("Artifact.xml") from the store

But we are not done yet, in fact this is just the begining. In the next tutorial we will package and precompile our hosting environment and build a base docker image to deploy our applications on.
We will then look at how we can use the manifest to reduce our deployed application size when we have a self contained application without the SDK installed in our image.

I welcome any feedback, if you found it useful, problems mistakes or improvements!
