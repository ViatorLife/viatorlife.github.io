[https://www.cnblogs.com/CKExp/p/17299198.html](https://www.cnblogs.com/CKExp/p/17299198.html)


## 前言

有时候想快速验证一些想法，新建一个控制台来弄，可控制台模板是轻量级的应用程序模板，不具备配置、日志、依赖注入等一些功能。


## 通用主机

在Asp.Net Core中有WebHostBuilder来提供DI，Configuratio，日志等功能，很是齐全，如果是在控制台中使用呢，或是使用的场景不单单是Web呢，比如测试项目，独立Job，桌面应用等，除了手动一个一个搭建外，也可以使用Generic HostBuilder，类似于WebHostBuilder，也提供了基础的功能，免于我们再去基建。

[https://source.dot.net/#Microsoft.Extensions.Hosting/HostBuilder.cs,148](https://source.dot.net/#Microsoft.Extensions.Hosting/HostBuilder.cs,148)

Generic HostBuilder是在.Net Core2.1时候提出的并被设计成能够在非Http模式和Http模式下都可以用。


## 安装Nuget包

```plain
Install-Package Microsoft.Extensions.Hosting
```


## 构建主机

开始使用Generic HostBuilder，可以手动实例化HostBuilder，

```plain
var hostBuilder = new HostBuilder();
```
或是调用Host中的扩展方法来完成创建。
```plain
var hostBuilder = Host.CreateDefaultBuilder(args)
```
实际上，扩展方法中是在手动实例化的基础上包装并做好了一些基础配置与服务注册，源码位置：
[https://source.dot.net/#Microsoft.Extensions.Hosting/Host.cs,53](https://source.dot.net/#Microsoft.Extensions.Hosting/Host.cs,53)

微软官网文档中也提及了CreateDefaultBuilder中完成的一些事情。

[https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host#default-builder-settings](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host#default-builder-settings)


### 服务注册

Generic HostBuilder很是方便我们去使用DI，在HostBuilder尚未执行Build方法前，先注册一些服务到容器中，此处注册个简单服务。


以订单服务类作为示例，增加接口类与实现类。

```plain
public interface IOrderService
{
    void AutoCancel();
}


public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;


    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }


    public void AutoCancel()
    {
        _logger.LogInformation("Start AutoCancel");
        Console.WriteLine("Order canceled!");
        _logger.LogInformation("Finished AutoCancel");
    }
}


```
注册到服务容器中，此处使用瞬时的生命周期。
```plain
var hostBuilder = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddTransient<IOrderService, OrderService>();
    });
```


### 实例化主机

调用HostBuilder的Build方法，将约定好的服务全部转换到服务容器中，完成主机实例化。

[https://source.dot.net/#Microsoft.Extensions.Hosting/HostBuilder.cs,148](https://source.dot.net/#Microsoft.Extensions.Hosting/HostBuilder.cs,148)

实例化的主机其实现的接口是IHost

![141934422_4a909f65-f71b-4181-b803-668182eb70a1](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141934422_4a909f65-f71b-4181-b803-668182eb70a1.png)
```plain
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddTransient<IOrderService, OrderService>();
    })
    .Build();
```


### 运行主机

如想要长时间运行(像是后台Job，订阅服务处理等都行)，可以执行扩展方法Run()，实际还是调用的IHost接口中的StartAsync方法。

[https://source.dot.net/#Microsoft.Extensions.Hosting.Abstractions/HostingAbstractionsHostExtensions.cs,63](https://source.dot.net/#Microsoft.Extensions.Hosting.Abstractions/HostingAbstractionsHostExtensions.cs,63)

```plain
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddTransient<IOrderService, OrderService>();
    })
    .Build();


host.Run();
```
控制台运行起来，可以看到启动初始化的日志记录，与Web启动时日志相似。
![141935861_240fc06c-d125-478d-93f6-402e4d999a57](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141935861_240fc06c-d125-478d-93f6-402e4d999a57.png)

### 服务获取

当主机实例化完毕后，如我们想直接获取服务，执行服务方法，完成一些功能，那可以无需Run，在构建主机完毕后就从主机中获取服务执行方法。

```plain
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddTransient<IOrderService, OrderService>();
    })
    .Build();


// host.Run();


using (var serviceScope = host.Services.CreateScope())
{
    var services = serviceScope.ServiceProvider;
    var orderService = services.GetRequiredService<IOrderService>();
    orderService.AutoCancel();
}
```
启动后，构造函数注入，日志记录等功能一应俱全，不用再做基建工作了。
![141936927_61983e2b-878d-41dc-a9af-c720364a0b62](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141936927_61983e2b-878d-41dc-a9af-c720364a0b62.png)

## 参考

[https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host)


>2023-04-08,望技术有成后能回来看见自己的脚步



