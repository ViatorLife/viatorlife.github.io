[https://www.cnblogs.com/CKExp/p/17179291.html](https://www.cnblogs.com/CKExp/p/17179291.html)


## 前言

有时候想快速验证一些想法，新建一个控制台来弄，可控制台模板是轻量级的应用程序模板，不具备配置、日志、依赖注入等一些功能。


## 依赖注入

在Asp.Net Core应用程序中，可以通过依赖注入使用IConfiguration接口来使用配置。而控制台模板十分简单，没有内置依赖注入，应用程序所依赖的功能(如配置)不容易获得。使用IOC(控制反转)容器实现依赖注入简要步骤

1. 创建Service Collection。

2. 将服务注册到Service Collection。

3. Service Collection构建Service Provider。

4. 使用Service Provider获取服务


## 安装Nuget包

```plain
Install-Package Microsoft.Extensions.DependencyInjection
```


## 创建服务容器

需要准备一个Service Collection(服务容器)，用来存储服务的注册与从其中获取服务。微软已提供了ServiceCollection类。

[https://source.dot.net/#Microsoft.Extensions.DependencyInjection.Abstractions/ServiceCollection.cs,4219db14266c9d57](https://source.dot.net/#Microsoft.Extensions.DependencyInjection.Abstractions/ServiceCollection.cs,4219db14266c9d57)

在Asp.Net Core 源码中，Service Collection在WebHostBuilder中实例化好了。

[https://source.dot.net/#Microsoft.AspNetCore.Hosting/WebHostBuilder.cs,c278c6966995108c](https://source.dot.net/#Microsoft.AspNetCore.Hosting/WebHostBuilder.cs,c278c6966995108c)

BuildCommonServices方法中，完成Service Collection实例化与应用需要的服务注册。

![141240531_694b6452-ad2d-461d-90df-06737c059531](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141240531_694b6452-ad2d-461d-90df-06737c059531.png)
在控制台模板中，需要自己去创建实例。

```plain
var services = new ServiceCollection();
```


## 服务注册

服务注册到服务容器中时需要指明服务本身的生命周期，有三种设置，全局单例(Singleton)，请求内单例(Scope)，瞬时(Transient)。以订单服务类作为示例，增加接口类与实现类。

```plain
public interface IOrderService
{
    void AutoCancel();
}


public class OrderService : IOrderService
{
    public void AutoCancel()
    {
        Console.WriteLine("Order canceled!");
    }
}
```
注册到服务容器中，此处使用瞬时的生命周期。
```plain
services.AddTransient<IOrderService, OrderService>();
```


## 构建服务提供者

通过服务容器转换得到服务提供者，使用已有扩展方法

[https://source.dot.net/#Microsoft.Extensions.DependencyInjection/ServiceCollectionContainerBuilderExtensions.cs,346262d4cea1139c](https://source.dot.net/#Microsoft.Extensions.DependencyInjection/ServiceCollectionContainerBuilderExtensions.cs,346262d4cea1139c)

```plain
using(var serviceProvider = services.BuildServiceProvider(true))
{
    //...
}
```


## 获取服务

可通过服务提供者开辟本次使用的一个范围，再获取IOrderService，从注册的服务中获取对应的服务实例。

```plain
using (var serviceProvider = services.BuildServiceProvider(true))
{
    var scope = serviceProvider.CreateScope();
    var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();
    
    orderService.AutoCancel();
}
```
如此一来，控制台模板下，也可以使用到服务容器来快速验证一些想法。

>2023-03-05,望技术有成后能回来看见自己的脚步

