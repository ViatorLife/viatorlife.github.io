[https://www.cnblogs.com/CKExp/p/17575349.html](https://www.cnblogs.com/CKExp/p/17575349.html)


### 前言

生活中可以见到很多传送带，河道，工厂流水线，快递服务等。去站点寄个快递，通过传送带，将快递从一端传递到另一端，再去站点收个快递。参照这种设计，我们可以将其融入到软件中，以实现许多功能。

在.Net Core中实现了一个高效，线程安全的队列System.Threading.Channels，与RabbitMQ、Kafka这种跨进程的消息队列，使用场景上不同，Channels在进程内跨线程使用。从使用方式上，Channels也更简单，不需要额外引入包(.Net Core3.0后)。

![002210891_aef657dc-aa14-4920-a2da-a820c9fccf94](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002210891_aef657dc-aa14-4920-a2da-a820c9fccf94.png)
### Channels

Channel其本质上是一个高效的、线程安全的队列，支持在生产者和消费者之间存储和传递数据。一端(生产者)负责将数据写入Channel，另一端(消费者)可以从Channel中读取数据并进行处理。

[https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)

**Channel机制下一个消息只能由一个消费者来消费。**


#### Channel的两种模式

* Bounded Channels: 有限存储空间的Channel，对传入消息的容量有限，限定生产者在满足空间数量之前只能发布特定的次数。超过了则需要等待(非阻塞等待)消费者消费减少空间内消息数量后再继续发布。

* Unbounded Channels: 无限存储空间的Channel，生产者可以一直发布，但如果消费者的消费跟不上节奏，那么资源则会不断增大，有可能耗尽服务器资源。两者都是通过[Channel](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/Channel.cs,7)提供的工厂方法来创建。

```plain
public static class Channel
{
    public static Channel<T> CreateBounded(int capacity);
    public static Channel<T> CreateBounded(BoundedChannelOptions options);
    public static Channel<T> CreateUnbounded();
    public static Channel<T> CreateUnbounded(UnboundedChannelOptions options);
}
```
对于不同的模式会有不同的实现类。
```plain
public abstract class Channel<TWrite, TRead>
{
    public ChannelReader<TRead> Reader { get; protected set; } = null!;
    public ChannelWriter<TWrite> Writer { get; protected set; } = null!;
}


public abstract class Channel<T> : Channel<T, T> { }


public class BoundedChannel<T> : Channel<T> { }


public class UnboundedChannel<T> : Channel<T> { }
```


#### 写入到Channel

生产者可以通过[ChannelWrite](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/ChannelWriter.cs,12)来将消息写入到Channel中。

```plain
public abstract class ChannelWriter<T>
{
    public abstract bool TryWrite(T item);
    public abstract ValueTask<bool> WaitToWriteAysnc();
    public virtual ValueTask WriteAsync(T item);
    public virtual bool TryComplete(Exception? error = null);
}
```
对于不同场景可以选用相应写入方法，如TryWrite当是有限容量的Channel，写满后再写则返回false，而使用WaitToWrite后则会不断等待到有空间后才返回。
使用方式上，通过Channel来调用，

```plain
await channel.Writer.WriteAsync("xxx");
await channel.Writer.TryWrite("xxx");
```
在BoundedChannel和UnboundedChannel中都实现了私有密封类并继承自[ChannelWriter](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/ChannelWriter.cs,12)的[BoundedChannelWriter](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/BoundedChannel.cs,298)和[UnboundedChannelWriter](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/SingleConsumerUnboundedChannel.cs,204)。
```plain
public abstract class ChannelWriter<T> { }
public class BoundedChannelWriter : ChannelWriter<T> { }
public class UnboundedChannelWriter : ChannelWriter<T> { }
```


#### 从Channel读取

消费者可以从Channel中读取消息，通过ChannelReader提供了几个方法

```plain
public abstract class ChannelReader<T>
{
    public abstract bool TryRead(out T item);
    public abstract ValueTask<bool> WaitToReadAsync();
    public virtual ValueTask<T> ReadAsync();
    public virtual bool TryPeek(out T item);
    public virtual IAsyncEnumerable<T> ReadAllAsync();
}
```
几个方法使用和用途上和Write类似，唯独TryPeek这个方法只是读取，但是不消费即不会从Channel中移除消息。在[BoundedChannelReader](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/BoundedChannel.cs,60)和[UnboundedChannelReader](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/SingleConsumerUnboundedChannel.cs,53)中各自继承了[ChannelReader](https://source.dot.net/#System.Threading.Channels/System/Threading/Channels/ChannelReader.cs,13)并完成了实现。
```plain
public abstract partial class ChannelReader<T> {}
public class BoundedChannelReader : ChannelReader<T> {}
public class UnboundedChannelReader : ChannelReader<T> {}
```


### 快速上手

此处，在控制台中快速应用下这个Channels，新建一个控制台，Channels在3.0中直接包含在框架中了，不用在额外安装包了。

#### 简单使用

创建一个无限容量的Channel，并循环写入10条信息，然后循环读取Channel。

```plain
using System.Threading.Channels;


var unboundedChannel = Channel.CreateUnbounded<string>();


for(int i=0; i < 10; i++)
{
    await unboundedChannel.Writer.WriteAsync($"Hello World {i}");
    await Task.Delay(1000);
}


while (await unboundedChannel.Reader.WaitToReadAsync())
{
    if (unboundedChannel.Reader.TryRead(out var message))
    {
        Console.WriteLine($"{DateTime.Now} Read Content: {message}");
    }
}
```
执行启动后等待一段时间写入到Channel完毕，便能够见到从Channel中读取到的信息。
![002212314_e482a9b0-20b3-4e75-a54e-115b49fceb7a](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002212314_e482a9b0-20b3-4e75-a54e-115b49fceb7a.png)
#### 多线程使用

再此基础上扩展下，实现一个线程中写入消息，另一个线程中消费消息。

```plain
using System.Threading.Channels;


var unboundedChannel = Channel.CreateUnbounded<string>();


 _ = Task.Factory.StartNew(async () =>
    {
        for (int i = 0; i < 10; i++)
        {
            await unboundedChannel.Writer.WriteAsync($"Hello World {i}");
            await Task.Delay(1000);
        }
    });


while (await unboundedChannel.Reader.WaitToReadAsync())
{
    if (unboundedChannel.Reader.TryRead(out var message))
    {
        Console.WriteLine($"{DateTime.Now} Read Content: {message}");
    }
}
```
再次启动便是间隔1秒下不断的输出信息。
![002213700_c7cee1da-730d-4fe8-b517-c30f587826ef](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002213700_c7cee1da-730d-4fe8-b517-c30f587826ef.png)

### 场景应用

此处设计一个应用场景，请求进入代理站点后转发到目标站点，在这其中通过Channels传输站点的请求信息，消费端写入到数据库中来实现流量录制。

![002214847_1790dea7-e9b5-4996-bcce-c2d32b7deb59](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002214847_1790dea7-e9b5-4996-bcce-c2d32b7deb59.png)
新建两个WebApi项目，其中一个不做处理，使用默认模板即可。另一个中增加Yarp包与后台服务。

1. 在Yarp中间件中，记录请求与响应信息，将信息写入到Channel中。

```plain
using System.Threading.Channels;
using YarpDemo.Host;


var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy().LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
builder.Services.AddSingleton(Channel.CreateUnbounded<TrafficContent>());
builder.Services.AddHostedService<TrafficHandlerHostedService>();


var app = builder.Build();
app.MapGet("/Ping", () => "Hello World!");
app.MapReverseProxy(proxyPipeline =>
{
    proxyPipeline.Use(async (context, next) =>
    {
        var trafficContent = new TrafficContent()
        {
            Host = context.Request.Host.ToString(),
            Method = context.Request.Method,
            Path = context.Request.Path.ToString(),
            RequestTime = DateTime.Now,
        };


        await next();
        trafficContent.StatusCode = context.Response.StatusCode;
        trafficContent.ResponseTime = DateTime.Now;


        var channel = context.RequestServices.GetService<Channel<TrafficContent>>();
        await channel!.Writer.WriteAsync(trafficContent);
    });
});
app.Run();
```
1. 在后台服务中，非阻塞等待获取信息，此处直接以日志形式输出，还可在此基础上做些功能，比如存表，无效请求过滤，特定前缀、头过滤等。

```plain
public class TrafficHandlerHostedService : BackgroundService
{
    private readonly ILogger<TrafficHandlerHostedService> _logger;
    private readonly Channel<TrafficContent> _channel;


    public TrafficHandlerHostedService(ILogger<TrafficHandlerHostedService> logger, Channel<TrafficContent> channel)
    {
        _logger = logger;
        _channel = channel;
    }


    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Traffic Handler Hosted Service running.");


        try
        {
            while (await _channel.Reader.WaitToReadAsync())
            {
                if (_channel.Reader.TryRead(out var trafficContent))
                {
                    _logger.LogWarning($"Traffic: {trafficContent.Host}-{trafficContent.Method}-{trafficContent.Path}-{trafficContent.RequestTime}-{trafficContent.ResponseTime}-{trafficContent.StatusCode}");
                }
            }
        }
        catch (ChannelClosedException)
        {
            _logger.LogError("Channel was closed");
        }


        _logger.LogInformation("Traffic Handler Hosted Service is stopping.");
    }
}
```
1. 配置代理转发，直接转发到另一个WebApi中。

```plain
"ReverseProxy": {
  "Routes": {
    "routeAll": {
      "ClusterId": "clusterWebApi",
      "Match": {
        "Path": "{**catch-all}"
      }
    }
  },
  "Clusters": {
    "clusterWebApi": {
      "Destinations": {
        "webapi": {
          "Address": "http://localhost:6000/"
        }
      }
    }
  }
}
```
1. 启动WebApi和Yarp工程，发起请求，可见流量日志输出。![002216070_ea404253-1500-47c1-af39-8d96ec824d63](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002216070_ea404253-1500-47c1-af39-8d96ec824d63.png)

这种场景下可再使用读取到的站点发起对其他站点的请求，在Api升级切换中对比Api功能是否异常这种场景挺不错。


### 参考资料

* [https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)

* [https://deniskyashif.com/2019/12/08/csharp-channels-part-1/](https://deniskyashif.com/2019/12/08/csharp-channels-part-1/)


>2023-07-23,望技术有成后能回来看见自己的脚步

