---
layout: post
title:  "ILogger.BeginScope affects only loggers created by the same LoggerFactory"
date:   2019-03-12 12:03:19 +0000
categories: dev 
---

**Applies to**: ASP.NET Core 2.2 and the ApplicationInsights ILogger provider (but may also apply to other providers)

I'll start with the conclusion, because somebody else worded it well [this article][ref1]: "Because logging scopes are pretty loosely defined within the ASP.net Core framework, it pays to always test how scopes are handled within the particular logging library you are using. There is no way for certain to say that scopes are even supported, or that they will be output in a way that makes any sense." 

That being said, it appears that in ASP.NET Core 2.2 (with logging set up using the default DI) `innerLogger` will not recognize the scope unless `outerLogger` was created by the same [LoggerFactory][LoggerFactory] instance.

{% highlight csharp %}
using (outerLogger.BeginScope("{contextprop}", "value"))
{
    outerLogger.LogInformation("Outer logger");
    innerLogger.LogInformation("Inner logger");
}
{% endhighlight %}

Both these loggers are instances of [Microsoft.Extensions.Logging.Logger][Logger] and are containers for the actual logger implementations which you've actually set up your DI, in my case [ApplicationInsightsLogger][AppInsightsLogger]. If you click the links and look at the `BeginScope` implementation of each of those classes you'll that see they use a `IExternalScopeProvider` which is indeed managed instance level by [LoggerFactory][LoggerFactoryCode]. 

How did I get into such trouble as to have my loggers built by different factory instances in my ASP.NET Core Pages app? Here's how:

{% highlight csharp %}
 public Startup(IHostingEnvironment env, IConfiguration config, 
        ILoggerFactory loggerFactory)
{% endhighlight %}

I created the `innerLogger` using the factory instance above, in the [Startup class][Startup] class, while simply letting [ASP.NET Core's DI][SetupLogging] (which apparently used a different factory instance) to inject the `outerLogger` into my page model as below:

{% highlight csharp %}
public class IndexModel : PageModel
{
    public IndexModel(ILogger<IndexModel> logger)
    {
...
{% endhighlight %}

This pickle I was in was a code smell so a bit of refactoring was needed to ensure that all of the loggers in my user code were instantiated and injected by DI, thus removing the need for receiving the `ILoggerFactory` in the constructor. But the lesson I took from here is that the ILogger API and its implementations are not yet in .NET Core 2.2 at a level of standardization that make programming against interfaces and changing providers as predictable an experience as I would like it to be. On the bright side, though, it seems that [BeginScope][Logger3.0] is being reimplemented in .NET CORE 3.  

[ref1]: https://dotnetcoretutorials.com/2018/04/12/using-the-ilogger-beginscope-in-asp-net-core/
[LoggerFactory]: https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.loggerfactory?view=aspnetcore-2.2
[Startup]: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-2.2
[Logger]: https://github.com/aspnet/Logging/blob/7c6a5235ec2a518c40eca96da9b19f8a96d73b56/src/Microsoft.Extensions.Logging/Logger.cs#L125
[AppInsightsLogger]: https://github.com/Microsoft/ApplicationInsights-dotnet-logging/blob/f9728ec758b67042c27498d645e5ad0c1b1bf8c0/src/ILogger/ApplicationInsightsLogger.cs#L53
[LoggerFactoryCode]: https://github.com/aspnet/Logging/blob/7c6a5235ec2a518c40eca96da9b19f8a96d73b56/src/Microsoft.Extensions.Logging/LoggerFactory.cs#L116
[SetupLogging]: https://github.com/Microsoft/ApplicationInsights-dotnet-logging/tree/master/src/ILogger
[Logger3.0]: https://github.com/aspnet/Logging/blob/master/src/Microsoft.Extensions.Logging/Logger.cs#L98