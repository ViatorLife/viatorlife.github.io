[https://www.cnblogs.com/CKExp/p/17216231.html](https://www.cnblogs.com/CKExp/p/17216231.html)


## 前言

有时候想快速验证一些想法，新建一个控制台来弄，可控制台模板是轻量级的应用程序模板，不具备配置、日志、依赖注入等一些功能。


## 日志

.Net Core自带了一个基础的logger框架Microsoft.Extensions.Logging提供记录日志功能，能够按日志不同级别记录日志信息(Information, Warning, Error等)。

|LogLevel|值|扩展方法|描述|
|:----|:----|:----|:----|
|Trace|0|[LogTrace](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.extensions.logging.loggerextensions.logtrace?view=dotnet-plat-ext-7.0&viewFallbackFrom=net-8.0)|追踪：包含最详细的信息。可能包含机密信息。 预设为停用，且不应该在生产环境中启用。|
|Debug|1|[LogDebug](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.extensions.logging.loggerextensions.logdebug?view=dotnet-plat-ext-7.0&viewFallbackFrom=net-8.0)|调试：开发调试并在调试工具中展示。 生产环境中小心使用，日志条数庞大。|
|Information|2|[LogInformation](https://docs.microsoft.com/zh-tw/dotnet/api/microsoft.extensions.logging.loggerextensions.loginformation?view=dotnet-plat-ext-3.1)|信息：跟踪程序执行的一般流程。 可能具有长期值。|
|Warning|3|[LogWarning](https://docs.microsoft.com/zh-tw/dotnet/api/microsoft.extensions.logging.loggerextensions.logwarning?view=dotnet-plat-ext-3.1)|警告：针对异常或或非预期的事件。 通常包含不会导致应用失败的错误或状况。|
|Error|4|[LogError](https://docs.microsoft.com/zh-tw/dotnet/api/microsoft.extensions.logging.loggerextensions.logerror?view=dotnet-plat-ext-3.1)|错误：发生无法处理的错误或异常。 这些记录表示目前的执行流程或任务失败，但不是整个程序失败。|
|Critical|5|[LogCritical](https://docs.microsoft.com/zh-tw/dotnet/api/microsoft.extensions.logging.loggerextensions.logcritical?view=dotnet-plat-ext-3.1)|严重：发生需要立即关注的失败。 例如：磁盘空间不足。|
|None|6||指定记录类别不应写入任何信息。|

.Net Core中使用[ILogger](https://source.dot.net/#Microsoft.Extensions.Logging.Abstractions/ILogger.cs,0976525f5d1b9e54)来记录日志，可以在不同模板应用上使用，如控制台应用，Asp.Net Core，WPF等，Asp.Net Core里使用十分简单，可以直接通过DI注册与获取到ILogger，但在控制台模式中，需要我们自己去配置。


## 安装Nuget包

新建控制台项目并安装Logger的包，此处使用日志输出渠道为Console中直接展示

```plain
Install-Package Microsoft.Extensions.Logging.Console
```
Logging基础包(Microsoft.Extensions.Logging)在Console已有依赖,此处不再额外安装。同时，Logging基础包中依赖了DI的基础包，使用上极其方便了。
![141916937_7d2d29d0-cf3d-4e44-9836-ce053de02f8b](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141916937_7d2d29d0-cf3d-4e44-9836-ce053de02f8b.png)

## 服务注册

将日志服务加入到服务容器中，此处设置Console作为日志输出渠道，还可以设置其他渠道。

[https://source.dot.net/#Microsoft.Extensions.Logging/LoggingServiceCollectionExtensions.cs](https://source.dot.net/#Microsoft.Extensions.Logging/LoggingServiceCollectionExtensions.cs)

额外注册了一个OrderJob，用来在OrderJob中输出日志，构建ServiceProvider获取容器提供的服务实例。

```plain
var services = new ServiceCollection();
services.AddLogging(configure => configure.AddConsole());
services.AddTransient<OrderJob>();
using (ServiceProvider serviceProvider = services.BuildServiceProvider())
{
    var orderJob = serviceProvider.GetService<OrderJob>()!;
    orderJob.Execute();
}
```
在OrderJob中通过构造函数注入使用Logger，也可以注入ILoggerFactory创建Logger
```plain
public class OrderJob
{
    private readonly ILogger<OrderJob> _logger;


    public OrderJob(ILogger<OrderJob> logger)
    {
        _logger = logger;
    }


    public void Execute()
    {
        _logger.LogInformation("OrderJob Started at {dateTime}", DateTime.UtcNow);
        //Business Logic START
        //Business logic END
        _logger.LogInformation("OrderJob  Ended at {dateTime}", DateTime.UtcNow);
    }
}
```
如上启动后会输出日志信息到控制台中
![141918557_505c298e-1063-47d5-bcec-e3c31cae0afd](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141918557_505c298e-1063-47d5-bcec-e3c31cae0afd.png)
如还设置了其他日志输出渠道，比如写入文件，写入数据库等，则日志记录一并会写入到这些渠道中。


## 组成部分

.Net Core的日志模型主要由三个核心对象构成，Logger、LoggerProvider和LoggerFactory。

![141919517_c67ce1b8-53ae-4e9c-bfd2-89f07da97ee7](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141919517_c67ce1b8-53ae-4e9c-bfd2-89f07da97ee7.png)
注册阶段，可将N个[LoggerProvider](https://source.dot.net/#Microsoft.Extensions.Logging.Abstractions/ILoggerProvider.cs,eb15860506d1dbeb)注册到全局[LoggerFactory](https://source.dot.net/#Microsoft.Extensions.Logging/LoggerFactory.cs,173b9b523cabe719)中。

![141920594_1994bdd0-93d3-401e-82e9-77d2c41fba64](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141920594_1994bdd0-93d3-401e-82e9-77d2c41fba64.png)
各个LoggerProvider有自己的Logger实现，可以按照已有实现扩展。

![141922276_5d75c232-5374-44fb-a39c-b516e7d2a0bf](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141922276_5d75c232-5374-44fb-a39c-b516e7d2a0bf.png)
使用日志服务时，源码内部从LoggerFactory创建[Logger对象](https://source.dot.net/#Microsoft.Extensions.Logging/Logger.cs,0b9109ca7ae5464c)(注意这个是Factory的)，其[创建过程](https://source.dot.net/#Microsoft.Extensions.Logging/Logger.cs,fdb90470ff3a62bd)中将LoggerFactory拥有的、已注册的N个Provider组合转换最终得到MessageLogger(这个是Provider中Logger转换过来的)。

![141923955_b626250d-ae3d-4a77-b4e3-51b32d5ed48d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141923955_b626250d-ae3d-4a77-b4e3-51b32d5ed48d.png)
写日志时循环N个MessageLogger(最终还是使用各Provider提供的Logger)，满足日志级别，则使用写入日志，不满足则下一个。

![141925357_8911f281-3ed6-4973-8a81-c4474fb048cf](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141925357_8911f281-3ed6-4973-8a81-c4474fb048cf.png)

## 参考

[https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-7.0](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-7.0)

[https://www.cnblogs.com/artech/p/inside-net-core-logging-2.html](https://www.cnblogs.com/artech/p/inside-net-core-logging-2.html)


>2023-03-15,望技术有成后能回来看见自己的脚步

