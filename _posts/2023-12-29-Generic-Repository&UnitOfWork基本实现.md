[https://www.cnblogs.com/CKExp/p/17935007.html](https://www.cnblogs.com/CKExp/p/17935007.html)


## 前言

在DbContext中已经具备了事务，对于多个实体的操作，能够在一个事务中保证。借助仓储在基于DbContext上的封装，我们能够更好的扩展复用。泛型仓储的使用又能简化对于基础功能的依赖，但是当现有事务范围不足以覆盖或是多个仓储操作，多次调用SaveChange后，整体的事务范围便发生了变化，不能形成一个整体了。


## 事务范围

### 单次SaveChange

![002337221_eaeb2932-dc19-4c34-a187-93c99f2aeba3](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002337221_eaeb2932-dc19-4c34-a187-93c99f2aeba3.png)
新增Order及OrderItem，两者本身都没问题的话，一次SaveChange可以全部成功。而当SaveChange完毕后，再出现异常(如下直接抛出异常test)，已有的事务已经提交完毕，无法回滚之前的操作了。

```cs
[HttpPost]
public async Task<int> CreateAsync(string name)
{
    var order = new Order()
    {
        Name = name,
        CreateDate = DateTime.UtcNow,
        OrderItems = new List<OrderItem>()
        {
            new OrderItem()
            {
                ProductId = 1,
                Amount = 1,
            }
        }
    };
    order = await _orderRepository.InsertAsync(order, autoSave: true);
    //throw new Exception("test");


    return order.Id;
}
```


### 多次SaveChange

类似于直接抛出事务，实现多次调用SaveChange，

![002338280_74256c5d-9af4-40f8-984c-d43901a2f6a4](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002338280_74256c5d-9af4-40f8-984c-d43901a2f6a4.png)
第一次执行正常，执行SaveChange后数据入库，但在第二次操作时出现失败，第二次的入库操作能够回滚，但是第一次的SaveChange却是还是执行成功，从整个操作上，缺失了一致性。

```plain
public async Task<int> CreateMultiAsync(string name)
{
    var order = new Order()
    {
        Name = name,
        CreateDate = DateTime.UtcNow,
        OrderItems = new List<OrderItem>()
        {
            new OrderItem()
            {
                ProductId = 1,
                Amount = 1,
            }
        }
    };
    order = await _orderRepository.InsertAsync(order, autoSave: true);


    var orderError = new Order()
    {
        Name = name,
        CreateDate = DateTime.Now,//数据库中限制只允许utc
        OrderItems = new List<OrderItem>()
        {
            new OrderItem()
            {
                ProductId = 1,
                Amount = 1,
            }
        }
    };
    orderError = await _orderRepository.InsertAsync(orderError, autoSave: true);


    return order.Id;
}
```
这种场景在结合领域事件，在handler中还执行一些仓储操作时很容易出现，当没有一个全局的事务保证下，不可避免有这种风险。

## 全局事务

### 事务延伸

[EFCore事务机制中](https://learn.microsoft.com/zh-cn/ef/core/saving/transactions)，可以通过DbContext开启一个事务，然后将所有数据库操作纳入其中。对于单次或是多次SaveChange，便可以合并成一个事务，形成一个整体。

![002339284_0f47be2e-dcc8-4ad3-9cf7-4fde2d73e226](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002339284_0f47be2e-dcc8-4ad3-9cf7-4fde2d73e226.png)
第一次执行SaveChange操作，虽然插入到库，但并未实际提交，不会在表中出现。而当第二次SaveChange如成功，则执行Commit, 两个操作顺利入库。如果发生异常，则第一次操作便是被回滚。

```plain
public async Task<int> CreateMultiAsync(string name)
{
    var dbContext = await _orderRepository.GetDbContextAsync();


    using (var transition = dbContext.Database.BeginTransaction())
    {
        var order = new Order()
        {
            Name = name,
            CreateDate = DateTime.UtcNow,
            OrderItems = new List<OrderItem>()
            {
                new OrderItem()
                {
                    ProductId = 1,
                    Amount = 1,
                }
            }
        };
        order = await _orderRepository.InsertAsync(order, autoSave: true);


        var orderError = new Order()
        {
            Name = name,
            CreateDate = DateTime.Now,//数据库中限制只允许utc
            OrderItems = new List<OrderItem>()
            {
                new OrderItem()
                {
                    ProductId = 1,
                    Amount = 1,
                }
            }
        };
        orderError = await _orderRepository.InsertAsync(orderError, autoSave: true);
        transition.Commit();


        return order.Id;
    }
}
```
如此一来，事务的范围便延伸起来，但麻烦总是接踵而至，想要一个统一的机制来写事务的开启，提交，而不像如上一样，需要时候写一节代码；并且当多个服务协作时，代码上还是无法形成一个整体的事务，总会充斥着各种嵌套的、平级的调用顺序。
```plain
ServiceX.Create()
{
  using (var transition = dbContext.Database.BeginTransaction())
  {
    // do something
    ServiceA.Create();
    ServiceB.Create();
    transition.Commit();
  }
}


ServiceA.Create()
{
  using (var transition = dbContext.Database.BeginTransaction())
  {
    // do something
    transition.Commit();
  }
}


ServiceB.Create()
{
  using (var transition = dbContext.Database.BeginTransaction())
  {
    // do something
    transition.Commit();
  }
}
```


### 工作单元

每次从仓储中取得DbContext开启事务，稍显麻烦，封装一层，加入工作单元概念，由它来管理事务的开启与提交。

>Unit Of Work：维护受业务事务影响的对象列表，并协调变化的写入和并发问题的解决。即管理对象的CRUD操作，以及相应的事务与并发问题等。是用来解决领域模型存储和变更工作，而这些数据层业务并不属于领域模型本身具有的。
>从其定义来看，DbContext本身便已是实现了Uow，只是当我们需要将其扩展到一个业务用例时，需要协调多个仓储，多个聚合根等，其工作单元的范围也不再局限于DbContext的一次操作了，而是要扩展到整个用例的范围。
#### IUnitOfWork定义

首先将dbContext的获取与事务提交的职责移交到IUnitOfWork中，定义如下抽象接口

```plain
public interface IUnitOfWork : IDisposable
{
    IDbContextTransaction BeginTransaction();
    void Commit();
}c
```
借助于依赖注入与DbContext的生命周期，UnitOfWork和Repository能够共用DbContext，如此依赖，代码能够简化
```plain
_unitOfWork.BeginTransaction();


var order = new Order();
order = await _orderRepository.InsertAsync(order, autoSave: true);


_unitOfWork.Commit();
```
再进一步考虑嵌套事务，如果被嵌套的事务中提前进行了Commit，那么导致后续的操作则会报错，因此被嵌套的事务要么不能写，要么要有方法使其知道是被嵌套中，然后令其失效。
```plain
ServiceX.Create()
{
  _unitOfWorkOuter.BeginTransaction();
  // do something
   ServiceA.Create();
   ServiceB.Create();
  _unitOfWorkOuter.Commit();
}


ServiceA.Create
{
  _unitOfWorkInner.BeginTransaction();
  // do something
  _unitOfWorkInner.BeginTransaction();
}


ServiceB.Create
{
  _unitOfWorkInner.BeginTransaction();
  // do something
  _unitOfWorkInner.BeginTransaction();
}
```
#### IUnitOfWorkManager定义

为了使其能够区分被嵌套的事务，需要能够在外层事务开启后，再注入内层事务时，能够外层事务已经存在，或许可以再包一层，来保存外层事务，增加一个IUnitOfWorkManager(参照ABP中代码实现)。

```plain
public interface IUnitOfWorkManager
{
    IUnitOfWork Current { get; }
    IUnitOfWork Begin();
}
```
为了实现内层事务，新增IUnitOfWorkInner，增加空实现
```plain
public class UnitOfWorkInner : IUnitOfWork
{
    private readonly IUnitOfWork _unitOfWorkOuter;


    public UnitOfWorkInner(IUnitOfWork unitOfWorkOuter)
    {
        _unitOfWorkOuter = unitOfWorkOuter;
    }


    public IDbContextTransaction BeginTransaction()
    {
        return _unitOfWorkOuter.BeginTransaction();
    }


    public void Commit()
    {
    }


    public void Dispose()
    {
    }
}
```
在UnitOfWorkManager中，根据是否存在外层事务来决定内存事务的实例。
```plain
public class UnitOfWorkManager : IUnitOfWorkManager
{
    private readonly IUnitOfWork _unitOfWork;


    public UnitOfWorkManager(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }


    public IUnitOfWork Begin()
    {
        var outerUow = Current;


        if (outerUow != null)
        {
            return new UnitOfWorkInner(outerUow);
        }


        var uow = _unitOfWork;
        uow.BeginTransaction();


        Current = uow;


        return uow;
    }


    public IUnitOfWork Current
    {
        get { return GetCurrentUow(); }
        set { SetCurrentUow(value); }
    }
    private static readonly AsyncLocal<LocalUowWrapper> AsyncLocalUow = new AsyncLocal<LocalUowWrapper>();
    private static IUnitOfWork GetCurrentUow()
    {
        var uow = AsyncLocalUow.Value?.UnitOfWork;
        if (uow == null)
        {
            return null;
        }


        return uow;
    }
    private static void SetCurrentUow(IUnitOfWork value)
    {
        lock (AsyncLocalUow)
        {
            if (AsyncLocalUow.Value?.UnitOfWork == null)
            {
                if (AsyncLocalUow.Value != null)
                {
                    AsyncLocalUow.Value.UnitOfWork = value;
                }


                AsyncLocalUow.Value = new LocalUowWrapper(value);
                return;
            }


            AsyncLocalUow.Value.UnitOfWork = value;
        }
    }
    private class LocalUowWrapper
    {
        public IUnitOfWork UnitOfWork { get; set; }


        public LocalUowWrapper(IUnitOfWork unitOfWork)
        {
            UnitOfWork = unitOfWork;
        }
    }
}
```
当再次使用到嵌套事务时候，便可以多次Begin得到外层事务与内层事务，内层事务不具备实际提交功能。
```plain
var unitOfWorkOuter = _unitOfWorkManager.Begin();
var order = new Order();
order = await _orderRepository.InsertAsync(order, autoSave: true);


var unitOfWorkInner = _unitOfWorkManager.Begin();
var orderLog = new OrderLog();
await _orderlOGRepository.InsertAsync(orderLog, autoSave: true);
unitOfWorkInner.Commit();


unitOfWorkOuter.Commit();
```
尽管能够解决嵌套事务，然而还能够简化一些工作量，在DDD中，应用层有承担着事务的开启提交职责，借助于AOP，可以将该部分职责在请求用例时自动启动，也就减少了手动unitOfWorkManager的使用。又或是，Get请求不需要开启事务，可以在AOP中判断Get请求时，不开启事务。

## 参考资料

* [https://learn.microsoft.com/zh-cn/ef/core/saving/transactions](https://learn.microsoft.com/zh-cn/ef/core/saving/transactions)

* [https://cloud.tencent.com/developer/article/1505237](https://cloud.tencent.com/developer/article/1505237)


>2023-12-29,望技术有成后能回来看见自己的脚步。

