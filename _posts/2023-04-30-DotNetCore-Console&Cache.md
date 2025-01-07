[https://www.cnblogs.com/CKExp/p/17364932.html](https://www.cnblogs.com/CKExp/p/17364932.html)


## 前言

有时候想快速验证一些想法，新建一个控制台来弄，可控制台模板是轻量级的应用程序模板，不具备配置、日志、依赖注入等一些功能。


## 缓存

在网站开发中，缓存无处不在，它能够极大地提高硬件和软件的运行速度。性能优化的第一步便是使用缓存，例如频繁的从数据库中读取，需要和底层IO交互，性能受限，如将常用数据加载到内存中，那么便可以极大的提升性能。


## Nuget包

缓存抽象包

```plain
Install-Package Microsoft.Extensions.Caching.Abstractions
```
如使用内存做缓存
```plain
Install-Package Microsoft.Extensions.Caching.Memory
```
如使用Redis做缓存
```plain
Install-Package Microsoft.Extensions.Caching.Redis
```
还有其他一些缓存包，此处省略

## 缓存使用

新建控制台项目，此处使用Memory包，简单存取一个字符串到内存中。

```plain
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;


var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddMemoryCache();
    })
    .Build();


using (var serviceScope = host.Services.CreateScope())
{
    var services = serviceScope.ServiceProvider;
    var cache = services.GetRequiredService<IMemoryCache>();
    cache.Set("key", "value");


    Console.WriteLine(cache.Get("key"));
}
```
>如上使用了通用主机包 Microsoft.Extensions.Hosting

### 过期策略

除了纯设置key-value值外还可以设置缓存的有效期，存在四种设置策略。

* 永不过期

* 绝对过期：指定过期时长再访问就失效。

* 滑动过期：指定过期时长内访问，过期时常最终截止时间延长。

* 绝对+滑动过期：滑动过期基础上再设置绝对过期，避免无限延长。


### 滑动过期

可通过MemoryCacheEntryOptions来设置过期策略，此处设置1.5秒滑动过期。

```plain
cache.Set("key", "value", new MemoryCacheEntryOptions()
{
    SlidingExpiration = TimeSpan.FromSeconds(1.5)
});
Console.WriteLine($"{DateTime.Now:HHmmss}_Init_{cache.Get("key")}");
Thread.Sleep(1000);
Console.WriteLine($"{DateTime.Now:HHmmss}_First_{cache.Get("key")}");
Thread.Sleep(2000);
Console.WriteLine($"{DateTime.Now:HHmmss}_Second_{cache.Get("key")}");
```
在1秒后访问，过期时长延长，sleep2秒后再次访问则失效。
![142036070_30843af9-4890-43b9-ab4b-7decdc5218bc](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142036070_30843af9-4890-43b9-ab4b-7decdc5218bc.png)

### 缓存回调

在缓存失效时还可以执行些回调逻辑。

```plain
var options = new MemoryCacheEntryOptions()
{
    SlidingExpiration = TimeSpan.FromSeconds(1.5),
};
options.RegisterPostEvictionCallback((key, value, reason, state) =>
{
    Console.WriteLine("key=" + key);
    Console.WriteLine("value=" + value);
    Console.WriteLine("reason=" + reason);
    Console.WriteLine("state=" + state);
}, "statestr");
```
当超过缓存时长，执行回调逻辑，此处将key value输出，可以看到尽管缓存失效获取对应值为空了，但是缓存回调中还是有值的展示，在这可以做些处理，例如如果是特定值，可以再次加入到缓存中。
![142036987_3b5e2e27-5256-4db2-bad6-4178033a5f20](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142036987_3b5e2e27-5256-4db2-bad6-4178033a5f20.png)
[PostEvictionCallback可以绑定多个回调方法](https://source.dot.net/#Microsoft.Extensions.Caching.Abstractions/MemoryCacheEntryExtensions.cs,132)，注意每个回调方法在失效后只会执行一次，多次访问缓存key，当扫描到失效后执行回调，后续访问缓存key都不会获取到缓存实例了。


## 源码分析

[IMemoryCache源码](https://source.dot.net/#Microsoft.Extensions.Caching.Abstractions/IMemoryCache.cs,11)中定义了三个方法对应增删查操作。

![142038012_375dfe69-7c23-4aa8-be8d-21058e120f1c](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142038012_375dfe69-7c23-4aa8-be8d-21058e120f1c.png)
在这基础上还有一些[扩展方法](https://source.dot.net/#Microsoft.Extensions.Caching.Abstractions/MemoryCacheExtensions.cs,8)在Caching.Abstraction包中。

![142039331_b414d354-2a51-4174-8a33-69b88e6b864d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142039331_b414d354-2a51-4174-8a33-69b88e6b864d.png)
在[IMemoryCache具体实现中](https://source.dot.net/#Microsoft.Extensions.Caching.Memory/MemoryCache.cs,21)，最终是存储在字典中，[源码位置](https://source.dot.net/#Microsoft.Extensions.Caching.Memory/MemoryCache.cs,641)

![142040848_00756398-a1de-4bf2-85e6-d2075620966a](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142040848_00756398-a1de-4bf2-85e6-d2075620966a.png)
存储的实际对象是[CacheEntry](https://source.dot.net/#Microsoft.Extensions.Caching.Memory/CacheEntry.cs,15)，在MemoryCache中，将自身与key作为参数封装到CacheEntry中

![142042148_b02495d0-37ce-4ef4-b5c6-9f7d753f301e](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142042148_b02495d0-37ce-4ef4-b5c6-9f7d753f301e.png)
对于缓存的一些设置，如缓存项大小，缓存有效期，访问时扫描时间，缓存失效回调等，则通过[MemoryCacheOptions](https://source.dot.net/#Microsoft.Extensions.Caching.Memory/MemoryCacheOptions.cs,10)完成，可以在单个缓存项设置也可以在[全局服务注册时](https://source.dot.net/#Microsoft.Extensions.Caching.Memory/MemoryCacheServiceCollectionExtensions.cs,41)设置好。

![142043203_b589c8bb-4f85-4656-9267-dffdecb56c7c](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/142043203_b589c8bb-4f85-4656-9267-dffdecb56c7c.png)

## 参考

[https://learn.microsoft.com/zh-cn/dotnet/core/extensions/caching](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/caching)


>2023-04-30,望技术有成后能回来看见自己的脚步

