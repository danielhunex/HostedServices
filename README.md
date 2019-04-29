# .Net Core DI Scope is not associated with Web Request

I had a chance to work with someone briefly who is a manager and as well as writes code. He was sort of "I know everything and everyone else does not or has to learn from me". So I had no the option of working independently following patterns and practices, my knowledge and experience. Either I had to provide documentation from external source as to why I did things in a certain way or got into argument. The first incident happened when I was given task on an existing a .Net Core hosted service ([HostedServices](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-2.2)) project. The entire application was using the default dependency injection from the .net core. However, in the particular project I was working on, objects were created manually. The code looks like as shown below

```csharp
 
public class HelloContext
{
   private ISomething _smh ; 
     void Foo(string key)
    {
       switch(key)
        {
            case "A":
            _smh = new SomethingA();
            case "B"
            _smh = new SomethingB();
            default:
            throw new Exception();
        }
        _smh.ShakingMyHead();
    }
}
```
As you can see, this piece of code violates the open close principle because of just an ego; If I have to include `SomethingC` implementation of `ISomething`, I have to modify this code. I asked the manager/developer, why they did it that way, and why they didn't use the ([IServiceScopeFactory](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.iservicescopefactory?view=aspnetcore-2.2)) to create those object dynamically. He told me that DI lifetime Scope in .net core is tied to Per Web Request and the particular hosted service has nothing to do with web request. So I was told to leave it as is and just modify it for new implementation. 
However, the Scope lifetime is not tied to a Web Request at all, you can use it in any unit of work which could be accessing database, create azure blobs, or even writing out to a console. Scope in .net core is a DI lifetime coordinated by ([IServiceScopeFactory](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.iservicescopefactory?view=aspnetcore-2.2)) which implements `IDisposable` and creates objects that are registered in the DI `ConfigureServices` as Scope (`AddScoped()`). You can use  `IServiceScopeFactory` in a `using` block to create a scope and get create a scoped object which will be disposed after the block and will be recreated when you get back to the block of code again. 

## Implementing Hosted Service with Strategy Pattern and Scoped DI

The purpose of this article is so to show you that you can use scoped DI objects in the generic host (`IHost`) without any web application/web host and how to implement a hosted background service with strategy pattern in .net core

### .Net core background services
In .net core, background services can be implemented using hosted services. A hosted service in .net core is just a class that implements [IHostedService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice?view=aspnetcore-2.2). Background services might do a number of tasks which could be scheduled tasks (timer based), queue based tasks etc.  A hosted service can be hosted using Web Host ([IWebHostBuilder](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.hosting.iwebhostbuilder?view=aspnetcore-2.2)) or Generic host ([IHostBuilder](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostbuilder?view=aspnetcore-2.2)) (.net core version 2.1 and above). In this article, we will be using the generic host without any web app associated with. If you are not familiar about hosted services, I would recommend reading [Background tasks with hosted services](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-2.2) before continuing reading this article but you can still skip it.

Implementations of IHostedService are registered them at the `ConfigureService()` method into the DI container. All those hosted services will be started and stopped along with the application. Therefore, a little extra caution should be taken when using DI in hosted service. For example, if you use a constructor injection and use a scoped service, it will automatically become singleton - will live the entire lifetime of the application.

The benefits of Scoped lifetime is to make an object short lived, create it when you use it and dispose it after. Scoped services make sense for services that hold resources, for example, a database connection context, a web socket, etc that you don't want to be singleton.
If you closely, look at the above code snippet, you can easily match it to fit the Strategy Pattern. The .net core hosted service can be efficiently implemented using a strategy pattern. In the following section, we will develop a hosted service using .net core with strategy pattern, using the default DI service provided by the framework. The complete code can be found at ([version-1](https://github.com/danielhunex/hostedservice-dotnetcore/tree/master/Version-1)) and ([version-2](https://github.com/danielhunex/hostedservice-dotnetcore/tree/master/Version-2)) on github.

>In computer programming, the strategy pattern (also known as the policy pattern) is a behavioral software design pattern that enables selecting an algorithm at runtime. Instead of implementing a single algorithm directly, code receives run-time instructions as to which in a family of algorithms to use. ([Wikipedia](https://en.wikipedia.org/wiki/Strategy_pattern))

As you can understand from the above strategy pattern definition, we can perfectly fit the above code snippet into this pattern, we just have to dynamically get the right algorithm ( the right implemention of ISomething). 