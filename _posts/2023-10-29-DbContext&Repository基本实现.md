[https://www.cnblogs.com/CKExp/p/17796196.html](https://www.cnblogs.com/CKExp/p/17796196.html)


### 前言

对于仓储模式，各有看法不同，直接使用DbContext简单方便，使用仓储模式扩展复用较好。受限于场景的差异，人员技能熟悉程度，交付时间，成本等选择哪种方式也有不同。


### Controller&DbContext

当需要快速设计一个访问数据库Demo时，顺手便是Controller+DbContext，当然还有其他更为简便的方式，但是这套模式在使用频次上更高。

![002319673_fb9d3cf4-1276-4dc1-a62a-3fba572f4339](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002319673_fb9d3cf4-1276-4dc1-a62a-3fba572f4339.png)
DbContext内部提供了常用的一些方法，直接操作DbContext很是方便，很自由，想怎么取怎么取，同时因为DbContext内部封装好了事务，因此多次变更一次提交之类的都很方便。

```plain
[ApiController]
[Route("[controller]")]
public class OrderController : ControllerBase
{
    private readonly AppDbContext _dbContext;


    public OrderController(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }


    [HttpGet("{id}")]
    public async Task<Order> GetAsync(int id)
    {
        return await _dbContext.Orders.AsNoTracking().FirstAsync(o => o.Id == id);
    }


    [HttpGet]
    public async Task<List<Order>> GetListAsync()
    {
        return await _dbContext.Orders.AsNoTracking().ToListAsync();
    }


    [HttpPost]
    public async Task<int> CreateAsync(string name)
    {
        var order = new Order()
        {
            Name = name,
            CreateDate = DateTime.UtcNow,
        };
        await _dbContext.Orders.AddAsync(order);
        await _dbContext.SaveChangesAsync();


        return order.Id;
    }


    [HttpPut("{id}")]
    public async Task<int> UpdateAsync(int id, string name)
    {
        var order = await _dbContext.Orders.FirstAsync(o=>o.Id == id);
        order.Name = name;
        await _dbContext.SaveChangesAsync();
        return order.Id;
    }


    [HttpDelete("{id}")]
    public async Task DeleteAsync(int id)
    {
        var order = await _dbContext.Orders.FirstAsync(o => o.Id == id);
        _dbContext.Orders.Remove(order);
        await _dbContext.SaveChangesAsync();
    }
}
```
可是使用方便的同时，有其他问题也需要考虑，如上代码如需要在其他Controller中查询，是不是又得写一遍，各种操作都得再来一遍，代码重复度很高。查询部分需要更加优化的查询性能等，受限EFCore的查询，需要更换到Dapper，那么纯查询操作的接口需要大面积改代码，相比下，引入仓储模式可能更加容易扩展，代码重用上也更佳。
![002320875_472515c8-cfa1-441e-86cd-8d7f491b3f2a](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002320875_472515c8-cfa1-441e-86cd-8d7f491b3f2a.png)
对于仓储模式结合DbContext，可以见到很多观点，有人认为多此一举，DbContext本身已经具备仓储功能及工作单元，没必要再在DbContext基础上再去封装。也有人认为需要封装，能够减少耦合，可扩展性更强。[微软官方文档](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core#using-a-custom-repository-versus-using-ef-dbcontext-directly)里也对此给了自己的看法，建议在复杂场景中使用仓储模式，以能更低成本的测试代码。


### 仓储模式

仓储模式(Repository Pattern)是一种设计模式，它在领域层和数据访问层(如实体框架核心/Dapper)之间协调数据。仓储隐藏了存储或检索数据所需的逻辑。因此，我们的应用程序不会关心我们使用什么类型的 ORM，因为使用EFCore、Dapper或是直接使用Ado.Net都是在仓储内部进行，甚至于说，在仓储中读写文件来记录数据，而对外而言，还是不变的操作，只是底层的存储介质，仓储内部的操作逻辑变更为文件形式。如此一来，在业务逻辑与底层实现之间，能够更清楚地分离关注点。

![002322334_f8583a4c-30a5-4534-bff5-c607858defd7](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002322334_f8583a4c-30a5-4534-bff5-c607858defd7.png)

### 实现基本仓储模式

在Controller&DbContext的基础上，封装一层Repository，实现基本的仓储模式。解决方案与项目搭建略过。

![002323432_09f5722a-f26a-4994-ad81-824b682a6e4e](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002323432_09f5722a-f26a-4994-ad81-824b682a6e4e.png)
* Api层承载项目运行。

* Domain层管理实体与仓储接口。

* Infrastructure层管理DbContext与仓储实现。


#### 仓储接口定义

按照Controller&DbContext中使用到的操作，实现对Order的仓储接口定义，暂忽略泛型仓储的考虑。

```plain
public interface IOrderRepository
{
    Task<Order> GetAsync(int id);
    Task<List<Order>> GetListAsync();
    Task<Order> InsertAsync(Order order);
    Task<Order> UpdateAsync(Order order);
    Task DeleteAsync(Order order);
}
```


#### 仓储接口的实现

实现过程则是对DbContext的操作，仅将Controller&DbContext中操作DbContext部分分离到Repository中。

```plain
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _dbContext;


    public OrderRepository(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }


    public async Task<Order> GetAsync(int id)
    {
        return await _dbContext.Orders.AsNoTracking().FirstAsync(o => o.Id == id);
    }


    public async Task<List<Order>> GetListAsync()
    {
        return await _dbContext.Orders.AsNoTracking().ToListAsync();
    }


    public async Task<Order> InsertAsync(Order order)
    {
        await _dbContext.Orders.AddAsync(order);
        await _dbContext.SaveChangesAsync();
        return order;
    }


    public async Task<Order> UpdateAsync(Order order)
    {
        _dbContext.Update(order);
        await _dbContext.SaveChangesAsync();
        return order;
    }


    public async Task DeleteAsync(Order order)
    {
        _dbContext.Orders.Remove(order);
        await _dbContext.SaveChangesAsync();
    }
}


```


#### Controller中使用仓储接口

如此一来，Controller中变更为对仓储门面的操作了。

```plain
[ApiController]
[Route("[controller]")]
public class OrderController : ControllerBase
{
    private readonly IOrderRepository _orderRepository;


    public OrderController(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }


    [HttpGet("{id}")]
    public async Task<Order> GetAsync(int id)
    {
        return await _orderRepository.GetAsync(id);
    }


    [HttpGet]
    public async Task<List<Order>> GetListAsync()
    {
        return await _orderRepository.GetListAsync();
    }


    [HttpPost]
    public async Task<int> CreateAsync(string name)
    {
        var order = new Order()
        {
            Name = name,
            CreateDate = DateTime.UtcNow,
        };
        order = await _orderRepository.InsertAsync(order);
        return order.Id;
    }


    [HttpPut("{id}")]
    public async Task<int> UpdateAsync(int id, string name)
    {
        var order = await _orderRepository.GetAsync(id);
        order.Name = name;
        order = await _orderRepository.UpdateAsync(order);
        return order.Id;
    }


    [HttpDelete("{id}")]
    public async Task DeleteAsync(int id)
    {
        var order = await _orderRepository.GetAsync(id);
        await _orderRepository.DeleteAsync(order);
    }
}
```


### 接口测试&Mock仓储

DBContext已经以非常少的努力代表了存储库和UoW（工作单元）实现。并且可以在代码的任何地方使用DBContext（从管道或中间件中依赖注入)，也可以直接使用。但是，如果想要能够执行测试，却是相对繁琐，使用上仓储模式后，再以单元测试模式来测试下接口。

```plain
public class OrderControllerUnitTest
{
    [Test]
    public async Task GetAllOrders_ShouldReturnAllOrders()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(repo => repo.GetListAsync())
            .ReturnsAsync(GetTestOrders());
        var controller = new OrderController(mockRepo.Object);


        // Act
        var result = await controller.GetListAsync();


        // Assert
        Assert.That(result.Count, Is.EqualTo(4));
    }


    [Test]
    public async Task GetOrderAsync_ShouldReturnOrder()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(repo => repo.GetAsync(It.IsAny<int>()))
            .ReturnsAsync(GetTestOrder());
        var controller = new OrderController(mockRepo.Object);


        // Act
        var result = await controller.GetAsync(1);


        // Assert
        Assert.That(result.Id, Is.EqualTo(1));
    }


    [Test]
    public void GetOrderAsync_ShouldThrowException()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        mockRepo.Setup(repo => repo.GetAsync(It.IsAny<int>()))
            .Throws(new Exception("The entity does not exist."));
        var controller = new OrderController(mockRepo.Object);


        // Assert,Act
        Assert.ThrowsAsync<Exception>(async () =>
        {
            await controller.GetAsync(0);
        });
    }


    #region MockData
    private List<Order> GetTestOrders()
    {
        var testOrders = new List<Order>
        {
            new Order { Id = 1, Name = "Demo1", CreateDate = DateTime.Now },
            new Order { Id = 2, Name = "Demo2", CreateDate = DateTime.Now },
            new Order { Id = 3, Name = "Demo3", CreateDate = DateTime.Now },
            new Order { Id = 4, Name = "Demo4", CreateDate = DateTime.Now }
        };
        return testOrders;
    }


    private Order GetTestOrder()
    {
        var testOrder = new Order { Id = 1, Name = "Demo1", CreateDate = DateTime.Now };
        return testOrder;
    }
    #endregion
}
```


### 参考资料

[https://codewithmukesh.com/blog/repository-pattern-in-aspnet-core/](https://codewithmukesh.com/blog/repository-pattern-in-aspnet-core/)

[https://www.thereformedprogrammer.net/is-the-repository-pattern-useful-with-entity-framework-core/](https://www.thereformedprogrammer.net/is-the-repository-pattern-useful-with-entity-framework-core/)

[https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application)

[https://procodeguide.com/programming/repository-pattern-in-aspnet-core/](https://procodeguide.com/programming/repository-pattern-in-aspnet-core/)

[https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy#repository-pattern](https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy#repository-pattern)

[https://learn.microsoft.com/en-us/ef/core/testing/testing-without-the-database#repository-pattern](https://learn.microsoft.com/en-us/ef/core/testing/testing-without-the-database#repository-pattern)


>2023-10-29,望技术有成后能回来看见自己的脚步。

