[https://www.cnblogs.com/CKExp/p/17863135.html](https://www.cnblogs.com/CKExp/p/17863135.html)


### 前言

自定义仓储能够很大程度方便我们实现功能，但是对于自定义仓储中的公共部分，又是非常基础的功能，如基础增删改和列表查询，分页查询，单个查询等，对于大部分自定义仓储来讲都能够用的上，如果每个自定义仓储中都实现一套，代码冗余度太高，无效工作过滤耗费时间。


### 构建泛型仓储

![002330855_a1128d23-4652-430a-97bc-30d481e40d7d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002330855_a1128d23-4652-430a-97bc-30d481e40d7d.png)

#### 泛型仓储抽象接口

在自定义仓储接口基础上将常用的CRUD抽离出来到泛型仓储接口中，原有自定义变成泛型来代替，以使得自定义仓储接口集成泛型仓储接口后仍能够有基础常用功能。

![002332303_ad9db6f5-be88-4bc5-87c3-cf7c2c733aff](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002332303_ad9db6f5-be88-4bc5-87c3-cf7c2c733aff.png)
对于自定义仓储的设计，如果实体能够映射到数据库表，那么便可为其增加仓储以便实现读写功能。因此，实体便是成为大多数自定义仓储命名规则，此处增加一个实体接口，一方面标时实体，另一方面约束泛型仓储，以只允许具有实体接口标时的实体才能使用泛型仓储。

```plain
public interface IEntity
{
}


public interface IEntity<TKey> : IEntity
{
    TKey Id { get; }
}
```
泛型仓储的设计上，参照ABP中泛型仓储，简化如下
```plain
public interface IRepository
{


}


public interface IRepository<TEntity, TKey> : IRepository
    where TEntity : class, IEntity<TKey>
{
    Task<TEntity> GetAsync(TKey id);
    Task<TEntity> GetAsync([NotNull] Expression<Func<TEntity, bool>> predicate);
    Task<TEntity> FindAsync(TKey id);
    Task<TEntity> FindAsync([NotNull] Expression<Func<TEntity, bool>> predicate);
    Task<List<TEntity>> GetListAsync();
    Task<List<TEntity>> GetListAsync([NotNull] Expression<Func<TEntity, bool>> predicate);
    Task<List<TEntity>> GetPagedListAsync(int skipCount, int maxResultCount, string sorting);
    Task<long> GetCountAsync();


    Task<TEntity> InsertAsync([NotNull] TEntity entity, bool autoSave = false);
    Task InsertManyAsync([NotNull] IEnumerable<TEntity> entities, bool autoSave = false);


    Task<TEntity> UpdateAsync([NotNull] TEntity entity, bool autoSave = false);
    Task UpdateManyAsync([NotNull] IEnumerable<TEntity> entities, bool autoSave = false);


    Task DeleteAsync(TKey id, bool autoSave = false);
    Task DeleteAsync([NotNull] TEntity entity, bool autoSave = false);
    Task DeleteAsync([NotNull] Expression<Func<TEntity, bool>> predicate, bool autoSave = false);
    Task DeleteManyAsync([NotNull] IEnumerable<TEntity> entities, bool autoSave = false);
    Task DeleteManyAsync([NotNull] IEnumerable<TKey> ids, bool autoSave = false);


    Task SaveChangesAsync();
    Task<IQueryable<TEntity>> WithDetailsAsync(params Expression<Func<TEntity, object>>[] propertySelectors);
    Task<IQueryable<TEntity>> GetQueryableAsync();
}
```


#### 泛型仓储抽象实现

对于泛型仓储实现，先实现一个RepositoryBase，实现中并没有涉及到具体的技术实现，而是泛型仓储实现的抽象。与泛型仓储可属于同一层，都是领域层。

```plain
public abstract class RepositoryBase<TEntity, TKey> : IRepository<TEntity, TKey>
    where TEntity : class, IEntity<TKey>
{
    public virtual async Task<TEntity> GetAsync(TKey id)
    {
        var entity = await FindAsync(id);


        if (entity == null)
        {
            throw new EntityNotFoundException(typeof(TEntity), id);
        }


        return entity;
    }
    public virtual async Task<TEntity> GetAsync(Expression<Func<TEntity, bool>> predicate)
    {
        var entity = await FindAsync(predicate);


        if (entity == null)
        {
            throw new EntityNotFoundException(typeof(TEntity));
        }


        return entity;
    }
    public abstract Task<TEntity> FindAsync(TKey id);
    public abstract Task<TEntity> FindAsync(Expression<Func<TEntity, bool>> predicate);
    public abstract Task<List<TEntity>> GetListAsync();
    public abstract Task<List<TEntity>> GetListAsync(Expression<Func<TEntity, bool>> predicate);
    public abstract Task<List<TEntity>> GetPagedListAsync(int skipCount, int maxResultCount, string sorting);
    public abstract Task<long> GetCountAsync();


    public abstract Task<TEntity> InsertAsync(TEntity entity, bool autoSave = false);
    public virtual async Task InsertManyAsync(IEnumerable<TEntity> entities, bool autoSave = false)
    {
        foreach (var entity in entities)
        {
            await InsertAsync(entity);
        }


        if (autoSave)
        {
            await SaveChangesAsync();
        }
    }


    public abstract Task<TEntity> UpdateAsync(TEntity entity, bool autoSave = false);
    public virtual async Task UpdateManyAsync(IEnumerable<TEntity> entities, bool autoSave = false)
    {
        foreach (var entity in entities)
        {
            await UpdateAsync(entity);
        }


        if (autoSave)
        {
            await SaveChangesAsync();
        }
    }


    public virtual async Task DeleteAsync(TKey id, bool autoSave = false)
    {
        var entity = await FindAsync(id);
        if (entity == null)
        {
            return;
        }


        await DeleteAsync(entity, autoSave);
    }
    public abstract Task DeleteAsync(TEntity entity, bool autoSave = false);
    public abstract Task DeleteAsync(Expression<Func<TEntity, bool>> predicate, bool autoSave = false);
    public virtual async Task DeleteManyAsync(IEnumerable<TEntity> entities, bool autoSave = false)
    {
        foreach (var entity in entities)
        {
            await DeleteAsync(entity);
        }


        if (autoSave)
        {
            await SaveChangesAsync();
        }
    }
    public virtual async Task DeleteManyAsync([NotNull] IEnumerable<TKey> ids, bool autoSave = false)
    {
        foreach (var id in ids)
        {
            await DeleteAsync(id);
        }


        if (autoSave)
        {
            await SaveChangesAsync();
        }
    }


    #region Common
    public virtual Task SaveChangesAsync()
    {
        return Task.CompletedTask;
    }


    public virtual Task<IQueryable<TEntity>> WithDetailsAsync(params Expression<Func<TEntity, object>>[] propertySelectors)
    {
        return GetQueryableAsync();
    }


    public abstract Task<IQueryable<TEntity>> GetQueryableAsync();
    #endregion
}
```
#### 泛型仓储实现

在基础设施层中，实现具体的泛型仓储。此处围绕着EFCore来。

```plain
public class EfCoreRepositoryBase<TDbContext, TEntity, TKey> : RepositoryBase<TEntity, TKey>
    where TDbContext : DbContext
    where TEntity : class, IEntity<TKey>
{
    private readonly TDbContext _dbContext;


    public EfCoreRepositoryBase(TDbContext dbContext)
    {
        _dbContext = dbContext;
    }


    public override async Task<TEntity> FindAsync(TKey id)
    {
        return await (await GetDbSetAsync()).FindAsync(new object[] { id });
    }
    public override async Task<TEntity> FindAsync(Expression<Func<TEntity, bool>> predicate)
    {
        return await (await GetDbSetAsync()).Where(predicate).SingleOrDefaultAsync();
    }
    public override async Task<List<TEntity>> GetListAsync()
    {
        return await (await GetDbSetAsync()).ToListAsync();
    }
    public override async Task<List<TEntity>> GetListAsync(Expression<Func<TEntity, bool>> predicate)
    {
        return await (await GetDbSetAsync()).Where(predicate).ToListAsync();
    }
    public override async Task<List<TEntity>> GetPagedListAsync(int skipCount,int maxResultCount,string sorting)
    {
        var queryable = await GetDbSetAsync();


        return await queryable
            .OrderByIf<TEntity, IQueryable<TEntity>>(!sorting.IsNullOrWhiteSpace(), sorting)
            .PageBy(skipCount, maxResultCount)
            .ToListAsync();
    }
    public override async Task<long> GetCountAsync()
    {
        return await (await GetDbSetAsync()).LongCountAsync();
    }


    public override async Task<TEntity> InsertAsync(TEntity entity, bool autoSave = false)
    {
        var dbContext = await GetDbContextAsync();


        var savedEntity = (await dbContext.Set<TEntity>().AddAsync(entity)).Entity;


        if (autoSave)
        {
            await dbContext.SaveChangesAsync();
        }


        return savedEntity;
    }


    public override async Task<TEntity> UpdateAsync(TEntity entity, bool autoSave = false)
    {
        var dbContext = await GetDbContextAsync();


        dbContext.Attach(entity);


        var updatedEntity = dbContext.Update(entity).Entity;


        if (autoSave)
        {
            await dbContext.SaveChangesAsync();
        }


        return updatedEntity;
    }


    public override async Task DeleteAsync(TEntity entity, bool autoSave = false)
    {
        var dbContext = await GetDbContextAsync();


        dbContext.Set<TEntity>().Remove(entity);


        if (autoSave)
        {
            await dbContext.SaveChangesAsync();
        }
    }
    public override async Task DeleteAsync(Expression<Func<TEntity, bool>> predicate, bool autoSave = false)
    {
        var dbContext = await GetDbContextAsync();
        var dbSet = dbContext.Set<TEntity>();


        var entities = await dbSet
            .Where(predicate)
            .ToListAsync();


        await DeleteManyAsync(entities, autoSave);


        if (autoSave)
        {
            await dbContext.SaveChangesAsync();
        }
    }


    #region Common
    public override async Task SaveChangesAsync()
    {
        await (await GetDbContextAsync()).SaveChangesAsync();
    }


    public override async Task<IQueryable<TEntity>> WithDetailsAsync(params Expression<Func<TEntity, object>>[] propertySelectors)
    {
        return IncludeDetails(
            await GetQueryableAsync(),
            propertySelectors
        );
    }


    private static IQueryable<TEntity> IncludeDetails(
        IQueryable<TEntity> query,
        Expression<Func<TEntity, object>>[] propertySelectors)
    {
        if (!propertySelectors.IsNullOrEmpty())
        {
            foreach (var propertySelector in propertySelectors)
            {
                query = query.Include(propertySelector);
            }
        }


        return query;
    }


    public override async Task<IQueryable<TEntity>> GetQueryableAsync()
    {
        return (await GetDbSetAsync()).AsQueryable();
    }


    public virtual async Task<TDbContext> GetDbContextAsync()
    {
        return await Task.FromResult((TDbContext)_dbContext);
    }


    protected async Task<DbSet<TEntity>> GetDbSetAsync()
    {
        return (await GetDbContextAsync()).Set<TEntity>();
    }
    #endregion
}
```


### 基于泛型仓储扩展自定义仓储

如果业务简单，直接使用泛型仓储便足够了，但如果泛型仓储不够能力了，原自定义仓储可以继承泛型仓储并扩展其想要的能力。

```plain
public interface IOrderRepository : IRepository<Order, int>
{
  Task<int> GetEmergencyOrderCount();
}
```
在基础设施层中，同样扩展泛型仓储实现，并在其中使用DbContext来构建需要的操作。
```plain
public class OrderRepository : EfCoreRepositoryBase<AppDbContext, Order, int>, IOrderRepository
{
  public async Task<int> GetEmergencyOrderCount()
  {
    var dbContext = await GetDbContextAsync();
    return await dbContext.Orders
      .AsNoTracking()
      .Where(o=>o.Status == 1)
      .CountAsync();
  }
}
```


### 泛型仓储服务注册

对于泛型仓储注册到DI容器中，对于自定义仓储或是其他服务，如果采用类似ABP中的全局扫描ITransientDependency接口方式自然不用关心。如果是挨个手动注册，那么对于自定义仓储或是泛型仓储来讲，可能存在遗漏，只能在出现没有注入实例下才会在手动注册到容器中。

```plain
builder.Services.AddTransient<IOrderRepository, OrderRepository>();
...


builder.Services.AddTransient<IRepository<Order, int>, EfCoreRepositoryBase<AppDbContext, Order, int>>();
...
```


#### 扫描程序集注册

如下简单实现扫描IEntity接口，为每一个实体注册泛型仓储（如果是应用了DDD体系，则建议是对聚合根实体注册泛型仓储，聚合的获取应当从聚合根开始）。也实现了扫描自定义仓储与实现注入到DI容器中以方便需要使用到自定义仓储。

```plain
public static class RepositoryExtensions
{
    public static IServiceCollection AddRepositories<TDbContext>(this IServiceCollection services, Assembly[] assemblies)
    {
        RegisterForGenericRepository<TDbContext>(services, assemblies);
        RegisterForCustomRepository(services, assemblies);


        return services;
    }


    private static void RegisterForGenericRepository<TDbContext>(IServiceCollection services, Assembly[] assemblies)
    {
        var entityTypes = assemblies
            .SelectMany(s => s.GetTypes())
            .Where(t => typeof(IEntity).IsAssignableFrom(t) && t != typeof(IEntity) && !t.IsGenericType)
            .ToArray();


        foreach (var entityType in entityTypes)
        {
            var primaryKeyType = FindPrimaryKeyType(entityType);
            var repositoryType = typeof(IRepository<,>).MakeGenericType(entityType, primaryKeyType);
            var efRepositoryType = typeof(EfCoreRepositoryBase<,,>).MakeGenericType(typeof(TDbContext), entityType, primaryKeyType);
            services.AddTransient(repositoryType, efRepositoryType);
        }
    }


    private static void RegisterForCustomRepository(IServiceCollection services, Assembly[] assemblies)
    {
        var interfaceTypes = assemblies
            .SelectMany(s => s.GetTypes())
            .Where(t => typeof(IRepository).IsAssignableFrom(t) && t != typeof(IRepository) && t.IsInterface && !t.IsGenericType)
            .ToArray();


        var implementTypes = assemblies
            .SelectMany(s => s.GetTypes())
            .Where(t => typeof(IRepository).IsAssignableFrom(t) && !t.IsInterface && !t.IsGenericType)
            .ToArray();


        foreach (var interfaceType in interfaceTypes)
        {
            var implementType = implementTypes.FirstOrDefault(interfaceType.IsAssignableFrom);
            if (implementType == null)
            {
                continue;
            }


            services.AddTransient(interfaceType, implementType);
        }
    }


    private static Type FindPrimaryKeyType([NotNull] Type entityType)
    {
        if (!typeof(IEntity).IsAssignableFrom(entityType))
        {
            throw new Exception($"Given {nameof(entityType)} is not an entity. It should implement {typeof(IEntity).AssemblyQualifiedName}!");
        }


        foreach (var interfaceType in entityType.GetTypeInfo().GetInterfaces())
        {
            if (interfaceType.GetTypeInfo().IsGenericType &&
                interfaceType.GetGenericTypeDefinition() == typeof(IEntity<>))
            {
                return interfaceType.GenericTypeArguments[0];
            }
        }


        return null;
    }
}
```
在服务配置时，便可以代替手动注册，通过扫描程序集方式完成服务批量注册
```plain
var assemblies = RuntimeHelper.GetAllAssemblies();
builder.Services.AddRepositories<AppDbContext>(assemblies.ToArray());
```


### 参考资料

[https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application#implement-a-generic-repository-and-a-unit-of-work-class](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application#implement-a-generic-repository-and-a-unit-of-work-class)


>2023-11-29,望技术有成后能回来看见自己的脚步。

