[https://www.cnblogs.com/CKExp/p/17737036.html](https://www.cnblogs.com/CKExp/p/17737036.html)


### 前言

获取当前请求用户的基础信息是很常见的，诸如当前用户Id，角色，有无访问权限等。通常我们可以直接使用HttpContext.User来拿到当前经过认证后的请求人信息。但是这样对于分层应用不太友好，需要安装AspNetCore.Http.Abstractions的包，这样对于这层(非Web层)来讲也有所侵入了。

### CurrentUser

很常用的一种方式是抽象与封装一层，比如ICurrentUser, ICurrentClient之类，具体实现放在Web层中以方便从HttpContext中获取用户或租户信息。而使用时，通过IOC在Domain中调用实例CurrentUser再获取详细信息。

![002312668_eff6f878-e07b-47b4-ac12-409a73d1e550](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002312668_eff6f878-e07b-47b4-ac12-409a73d1e550.png)
#### 代码示例

新建一个Asp.Net Core WebApi项目并增加一个Domain类库以添加ICurrentUser。

```plain
namespace CurrentUserDemo.Domain.CurrentUsers;


public interface ICurrentUser
{
    int? GetUserId();
    string? GetUserName();
}
```
在WebApi中实现CurrentUser，注入IHttpContextAccessor来获取当前用户信息。
```plain
using CurrentUserDemo.Api.Extensions;
using CurrentUserDemo.Domain.CurrentUsers;


namespace CurrentUserDemo.Api.Infrastructures.CurrentUsers;


public class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor;


    public CurrentUser(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }


    public int? GetUserId()
    {
        return _httpContextAccessor.HttpContext?.User?.FindUserId();
    }


    public string? GetUserName()
    {
        return _httpContextAccessor.HttpContext?.User?.FindUserName();
    }
}
```
当Domain层想要获取用户信息时，注入ICurrentUser。
```plain
using CurrentUserDemo.Domain.CurrentUsers;


namespace CurrentUserDemo.Domain.OrderServices;


public class OrderService
{
    private readonly ICurrentUser _currentUser;


    public OrderService(ICurrentUser currentUser)
    {
        _currentUser = currentUser;
    }


    public string Create(string name)
    {
        Console.WriteLine(name);
        return $"{name}:{_currentUser.GetUserId()!.Value}";
    }
}
```
如此一来，隔离了具体的Web实现，但也存在一些局限，如当后台任务(处于Web层或同级)调用非Web层时，如果在非Web层中想要获取用户信息或租户信息，那么便会报错，因为已实现的CurrentUser需要从HttpContext中获取信息，而后台任务并不走中间件，自然也没有HttpContext可言。为了满足这种情形，可以在CurrentUser实现与HttpContextAccessor之间再隔离一层用于场景适配。
### PrincipalAccessor

为了适配Job场景下使用CurrentUser获取用户信息，按如下方式再包一层来适配。

![002313718_4802c553-4894-43de-86d6-0a4cdccc8301](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002313718_4802c553-4894-43de-86d6-0a4cdccc8301.png)
此处适配这一层时还需注意后台任务是没有User的，因此，后台任务需要设置好默认用户或是走接口获取到用户信息，再要填充到PrincinalAccessor中，如此为了方便ICurrentUser获取用户信息时不至于获取失败而报错。

#### 代码示例

在如上小节代码基础上在WebApi层增加ICurrentPrincipalAccessor(如想作为基础包封装，可以将如上类全都抽出来做包，但需要依赖Http.Abstractions，也可以除HttpContextCurrentPrincipalAccessor外其余类抽出来做包，便没有外部包依赖了)。如下PrincipalAccessor实现过程借鉴于Abp源码(宝藏甚多，阅读不止)。

```plain
using System.Security.Claims;


namespace CurrentUserDemo.Api.Infrastructures.Claims;


public interface ICurrentPrincipalAccessor
{
    ClaimsPrincipal Principal { get; }


    IDisposable Change(ClaimsPrincipal principal);
}


```
其实现如下，在这其中，Change方法的目的在于允许手动设置当前用户的信息，使用AsyncLocal存储。
```plain
using CurrentUserDemo.Api.Infrastructures.Utilities;
using System.Security.Claims;


namespace CurrentUserDemo.Api.Infrastructures.Claims;


public abstract class CurrentPrincipalAccessorBase : ICurrentPrincipalAccessor
{
    public ClaimsPrincipal Principal => _currentPrincipal.Value ?? GetClaimsPrincipal();


    private readonly AsyncLocal<ClaimsPrincipal> _currentPrincipal = new AsyncLocal<ClaimsPrincipal>();


    protected abstract ClaimsPrincipal GetClaimsPrincipal();


    public virtual IDisposable Change(ClaimsPrincipal principal)
    {
        return SetCurrent(principal);
    }


    private IDisposable SetCurrent(ClaimsPrincipal principal)
    {
        var parent = Principal;
        _currentPrincipal.Value = principal;


        return new DisposeAction<ValueTuple<AsyncLocal<ClaimsPrincipal>, ClaimsPrincipal>>(static (state) =>
        {
            var (currentPrincipal, parent) = state;
            currentPrincipal.Value = parent;
        }, (_currentPrincipal, parent));
    }
}
```
再实现不同场景下的PrincipalAccessor，此处以Job和Http请求为例。
* 后台任务场景下使用

```plain
using System.Security.Claims;


namespace CurrentUserDemo.Api.Infrastructures.Claims;


public class ThreadCurrentPrincipalAccessor : CurrentPrincipalAccessorBase
{
    protected override ClaimsPrincipal GetClaimsPrincipal()
    {
        return Thread.CurrentPrincipal as ClaimsPrincipal;
    }
}
```
* Http请求场景下使用

```plain
using System.Security.Claims;


namespace CurrentUserDemo.Api.Infrastructures.Claims;


public class HttpContextCurrentPrincipalAccessor : ThreadCurrentPrincipalAccessor
{
    private readonly IHttpContextAccessor _httpContextAccessor;


    public HttpContextCurrentPrincipalAccessor(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }


    protected override ClaimsPrincipal GetClaimsPrincipal()
    {
        return _httpContextAccessor.HttpContext?.User ?? base.GetClaimsPrincipal();
    }
}
```
如此一来，在Job中获取用户信息时，先走Change，后再服务中便可以正常使用了。如下以Cap订阅消息后填充UserId为例。
```plain
public class MyCapFilter : SubscribeFilter
{
    private readonly ICurrentPrincipalAccessor _currentPrincipalAccessor;
    private IDisposable currentPrincipalAccessorDisposable;


    public MyCapFilter(ICurrentPrincipalAccessor currentPrincipalAccessor)
    {
        _currentPrincipalAccessor = currentPrincipalAccessor;
    }


    public override Task OnSubscribeExecutingAsync(ExecutingContext context)
    {
        var header = (context.Arguments.Last() as CapHeader)!;
        currentPrincipalAccessorDisposable = _currentPrincipalAccessor.Change(new Claim("uid", header["my.header.uid"] + "1"));
        return Task.CompletedTask;
    }
}
```
当消息订阅后，先填充到CurrentPrincipalAccessor中，在具体的Handler中，只需要直接注入ICurrentUser或是相关关联服务即可，如下对于OrderService的调用没有变化，OrderService本身也没有代码改动，只改动WebApi层即可让Job也能够调用非Web层代码。
```plain
using CurrentUserDemo.Domain.CurrentUsers;
using CurrentUserDemo.Domain.OrderServices;
using DotNetCore.CAP;


namespace CurrentUserDemo.Api.EventHandlers;


public class OrderCreatedEventHandler : ICapSubscribe
{
    private readonly ICurrentUser _currentUser;
    private readonly OrderService _orderService;


    public OrderCreatedEventHandler(ICurrentUser currentUser, OrderService orderService)
    {
        _currentUser = currentUser;
        _orderService = orderService;
    }


    [CapSubscribe("test.show.time")]
    public void ReceiveMessage(DateTime time, [FromCap] CapHeader header)
    {
        Console.WriteLine($"Current user id is: {_currentUser.GetUserId()}");
        Console.WriteLine($"Current order title: {_orderService.Create(name: header["my.header.orderName"]!)}");
    }
}
```
#### 总结

为了能获取用户信息，抽象了HttpContext.User来保存当前请求用户；为了实现不同层级能够使用用户或租户信息，抽象了ICurrentUser/ICurrentClient隔离；为了实现不同场景下(Job/Http请求下)统一对CurrentUser的调用，抽象了ICurrentPrincipalAccessor隔离。包一层以隔离变化，以适配场景。

>2023-09-29,望技术有成后能回来看见自己的脚步

