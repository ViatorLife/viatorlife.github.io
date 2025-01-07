[https://www.cnblogs.com/CKExp/p/17664069.html](https://www.cnblogs.com/CKExp/p/17664069.html)


在软件开发中，事件早已被我们所熟悉，一个按钮按下，产生中断事件，一个回车，前端页面有侦听事件，在事件风暴建模活动中，事件也是作为领域建模的突破口，事件的重要性不言而喻。其本质是发生的事实到引发了相关事情，在这其中的传递的信息便是事件的内容。就如同猫叫了，引发着老鼠跑了，主人醒了，其中的事件便是猫叫了，而该事件是猫执行叫的动作后的结果。


#### 领域事件

在领域驱动设计中，最开始的版本中并没有领域事件的概念，在 DDD 社区对领域驱动设计的内容不断的充实中，引入了领域事件。领域事件的命名遵循英语中的“名词 + 动词过去分词”格式，如，提交订单后发布的 OrderCreated 事件，订单完成后 OrderCompleted 事件，用以表示我们建模的领域中发生过的一件事情，也符合着事件本身是具有时间特征。对于事件所通知到的范围的不同，领域事件本身也分为两类：

* 领域内事件，即领域事件，作用于进程内的通信。

* 领域间事件，即集成事件，作用于进程间的通信。![002223749_bd25652f-0d4d-4b91-ba2f-3474e71062da](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002223749_bd25652f-0d4d-4b91-ba2f-3474e71062da.png)

为实现事件的发布，可以借助一些现有的包集成到系统中，但又为了隔离使用上依赖了具体的技术实现，我们可以对其进行一定的封装。


#### MediatR

MediatR是.NET中的开源简单中介者模式实现，它通过一种进程内消息传递机制（无其他外部依赖），进行请求/响应、命令、查询、通知和事件的消息传递，并通过泛型来支持消息的智能调度。

[https://github.com/jbogard/MediatR](https://github.com/jbogard/MediatR)


#### Cap

CAP 本身就是一个EventBus，能处理进程内消息，同时也是一个在微服务或者SOA系统中解决分布式事务问题的一个框架。它有助于创建可扩展，可靠并且易于更改的微服务系统。

[https://github.com/dotnetcore/CAP](https://github.com/dotnetcore/CAP)


#### 抽象EventBus

新建EventBus.Abstractions类库，用来抽象领域事件和集成事件的发布。

```plain
dotnet new classlib --name EventBusDemo.EventBus.Abstractions
```
在MediatR中，有一个[抽象的Contracts](https://github.com/jbogard/MediatR/tree/master/src/MediatR.Contracts)类库，方便我们构建一个抽象的EventBus。
![002225239_f35fd357-5ab4-44db-904c-d2778b4f6f07](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002225239_f35fd357-5ab4-44db-904c-d2778b4f6f07.png)
MediatR发布事件时，事件类上必须要有标记，比如如下INotification标记，因此，此处增加领域事件接口约束，来隔离INotification。

```plain
using MediatR;


namespace EventBusDemo.EventBus.Abstractions.Local;


public interface IDomainEvent : INotification
{
}
```
本地事件EventBus, 参照[MediatR中已有接口](https://github.com/jbogard/MediatR/blob/master/src/MediatR/IPublisher.cs)，替换约束类。
```plain
namespace EventBusDemo.EventBus.Abstractions.Local;


public interface ILocalEventBUs
{
    Task Publish(IDomainEvent domainEvent, CancellationToken cancellationToken = default);


    Task Publish<T>(T domainEvent, CancellationToken cancellationToken = default)
        where T : IDomainEvent;
}
```
同样，增加集成事件，Cap框架没有限制集成事件类的约束条件，而为了能够统一标识项目中的集成事件，还是有必要设置一个接口约束。
```plain
namespace EventBusDemo.EventBus.Abstractions.Distributed;


public interface IIntegrationEvent
{
}
```
增加集成事件的EventBus，参照[Cap框架中发布时的接口](https://github.com/dotnetcore/CAP/blob/master/src/DotNetCore.CAP/ICapPublisher.cs)并用IIntegrationEvent来约束事件。
```plain
namespace EventBusDemo.EventBus.Abstractions.Distributed;


public interface IDistrbutedEventBus
{
    Task PublishAsync<T>(string name, T? contentObj, string? callbackName = null, CancellationToken cancellationToken = default)
        where T : IIntegrationEvent;


    Task PublishAsync<T>(string name, T? contentObj, IDictionary<string, string?> headers, CancellationToken cancellationToken = default)
        where T : IIntegrationEvent;


    void Publish<T>(string name, T? contentObj, string? callbackName = null)
        where T : IIntegrationEvent;


    void Publish<T>(string name, T? contentObj, IDictionary<string, string?> headers)
        where T : IIntegrationEvent;


    Task PublishDelayAsync<T>(TimeSpan delayTime, string name, T? contentObj, IDictionary<string, string?> headers, CancellationToken cancellationToken = default)
        where T : IIntegrationEvent;


    Task PublishDelayAsync<T>(TimeSpan delayTime, string name, T? contentObj, string? callbackName = null, CancellationToken cancellationToken = default)
        where T : IIntegrationEvent;


    void PublishDelay<T>(TimeSpan delayTime, string name, T? contentObj, IDictionary<string, string?> headers)
        where T : IIntegrationEvent;


    void PublishDelay<T>(TimeSpan delayTime, string name, T? contentObj, string? callbackName = null)
        where T : IIntegrationEvent;
}
```


#### 实现EventBus

新建一个EventBus类库，来集成Cap和MediatR并实现EventBus.Abstractions。

```plain
dotnet new classlib --name EventBusDemo.EventBus
```
增加MediatR和Cap的包，Cap需要的具体存储和传输，可以在启动项目中配置。
```plain
<ItemGroup>
  <PackageReference Include="DotNetCore.CAP" Version="7.2.0" />
  <PackageReference Include="MediatR" Version="12.1.1" />
</ItemGroup>


<ItemGroup>
  <ProjectReference Include="..\EventBusDemo.EventBus.Abstractions\EventBusDemo.EventBus.Abstractions.csproj" />
</ItemGroup>
```
实现ILocalEventBus，实则时注入MediatR，将领域事件交给MediatR处理。
```plain
using EventBusDemo.EventBus.Abstractions.Local;
using MediatR;


namespace EventBusDemo.EventBus.Local;


public class LocalEventBus : ILocalEventBUs
{
    private readonly IPublisher _publisher;


    public LocalEventBus(IPublisher publisher)
    {
        _publisher = publisher;
    }


    public async Task Publish(IDomainEvent domainEvent, CancellationToken cancellationToken = default)
    {
        await _publisher.Publish(domainEvent, cancellationToken);
    }


    public async Task Publish<T>(T domainEvent, CancellationToken cancellationToken = default) where T : IDomainEvent
    {
        await _publisher.Publish(domainEvent, cancellationToken);
    }
}
```
实现IDistributedEventBus，也是注入Cap，将集成事件交给Cap处理。
```plain
using DotNetCore.CAP;
using EventBusDemo.EventBus.Abstractions.Distributed;


namespace EventBusDemo.EventBus.Distributed
{
    public class DistributedEventBus : IDistrbutedEventBus
    {
        private readonly ICapPublisher _capPublisher;


        public DistributedEventBus(ICapPublisher capPublisher)
        {
            _capPublisher = capPublisher;
        }


        public void Publish<T>(string name, T? contentObj, string? callbackName = null) where T : IIntegrationEvent
        {
            _capPublisher.Publish(name, contentObj, callbackName);
        }


        public void Publish<T>(string name, T? contentObj, IDictionary<string, string?> headers) where T : IIntegrationEvent
        {
            _capPublisher.Publish(name, contentObj, headers);
        }


        public async Task PublishAsync<T>(string name, T? contentObj, string? callbackName = null, CancellationToken cancellationToken = default) where T : IIntegrationEvent
        {
            await _capPublisher.PublishAsync(name, contentObj, callbackName, cancellationToken);
        }


        public async Task PublishAsync<T>(string name, T? contentObj, IDictionary<string, string?> headers, CancellationToken cancellationToken = default) where T : IIntegrationEvent
        {
            await _capPublisher.PublishAsync(name, contentObj, headers, cancellationToken);
        }


        public void PublishDelay<T>(TimeSpan delayTime, string name, T? contentObj, IDictionary<string, string?> headers) where T : IIntegrationEvent
        {
            _capPublisher.PublishDelay(delayTime, name, contentObj, headers);
        }


        public void PublishDelay<T>(TimeSpan delayTime, string name, T? contentObj, string? callbackName = null) where T : IIntegrationEvent
        {
            _capPublisher.PublishDelay(delayTime, name, contentObj, callbackName);
        }


        public async Task PublishDelayAsync<T>(TimeSpan delayTime, string name, T? contentObj, IDictionary<string, string?> headers, CancellationToken cancellationToken = default) where T : IIntegrationEvent
        {
            await _capPublisher.PublishDelayAsync(delayTime, name, contentObj, headers, cancellationToken);
        }


        public async Task PublishDelayAsync<T>(TimeSpan delayTime, string name, T? contentObj, string? callbackName = null, CancellationToken cancellationToken = default) where T : IIntegrationEvent
        {
            await _capPublisher.PublishDelayAsync(delayTime, name, contentObj, callbackName, cancellationToken);
        }
    }
}
```
如此一来，在Domain或是Application中使用IEventBus时，对底层具体使用的包便无感知(MediatR的服务注册和Cap的配置部分还可抽象一层来隔离)。
![002226735_0d5748ed-efaf-4ae5-b9ac-54a057d81767](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002226735_0d5748ed-efaf-4ae5-b9ac-54a057d81767.png)

#### 发布领域事件

如在DomainService中，将事件发布出去，通过观察者模式，找到对应的处理器，完成其他业务逻辑，以此来达到代码解耦。此处直接在服务中调用localEventBus使用，另外一种方案是在DbContext保存时通过SaveChange方法中来调用，这种情况需要事件先保存在某个位置，如聚合本身，后续文章在提到事务时一并扩展。

```plain
public class OrderService
{
    private readonly ILocalEventBus _localEventBus;


    public OrderService(ILocalEventBus localEventBus)
    {
        _localEventBus = localEventBus;
    }


    public async Task<Order> CreateOrder(string name)
    {
        var order = new Order()
        {
            Id = Guid.NewGuid(),
            Name = name,
        };


        await _localEventBus.Publish(new OrderCreatedDomainEvent()
        {
            Id = order.Id,
            Name = order.Name
        });


        return order;
    }
}
```


#### 发布集成事件

如在AppService中需要通过集成事件对外通知时，注入IDistributedEventBus，便可通过底层Cap将事件广播出去，由此实现发布订阅模式。此处对于事件名还可以融合到事件本身，后续文章中扩展。

```plain
public class OrderAppService
{
    private readonly IDistributedEventBus _distributedEventBus;
    private readonly OrderService _orderService;


    public OrderAppService(
        IDistributedEventBus distributedEventBus,
        OrderService orderService)
    {
        _distributedEventBus = distributedEventBus;
        _orderService = orderService;
    }


    public async Task<Guid> CreateOrder(string name)
    {
        var order = await _orderService.CreateOrder(name);


        await _distributedEventBus.PublishAsync("OrderCreated", new OrderCreatedIntegrationEvent()
        {
            Id = order.Id,
            Name = name
        });


        return order.Id;
    }
}
```


>2023-08-29,望技术有成后能回来看见自己的脚步

