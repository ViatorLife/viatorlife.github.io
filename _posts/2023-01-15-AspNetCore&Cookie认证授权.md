[https://www.cnblogs.com/CKExp/p/17053497.html](https://www.cnblogs.com/CKExp/p/17053497.html)


有时想快速搭建一个简单Demo或是需要验证授权完成后的一些动作，总是需要去找一番，有时还要不断去翻找到适合的，或是copy过来又不能使用又或是过时的。


认证与授权说来说去还是四个核心步骤

* 登录(SignIn)，为了获得当前请求人是谁的标识

* 退出(SignOut)，令系统移除请求人的标识

* 请求资源时识别请求人是谁(Authentication)

* 判断对资源请求的权限(Authorization)

   * 有身份信息但没有权限后拒绝操作(Challenge)

   * 没有身份信息则说未授权(Forbid)

   * 有身份信息且有权限则访问资源

![141106428_872fb2fd-157d-4228-bc38-7fe7a38e8e37](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141106428_872fb2fd-157d-4228-bc38-7fe7a38e8e37.png)
### Cookie

Cookie 是用来在客户端存储会话信息的，它要求服务器在响应给客户端信息时，携带一个头部字段 Set-Cookie，其内容为键值对，保存一些需要存储在客户端本地的信息。在之后的每一次请求中，浏览器会将这些信息，携带在请求头部字段 Cookie 中，发送给服务器，用以标识客户端的身份。

![141108238_8d98ca85-5f69-40a9-b45a-3d99121ec395](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141108238_8d98ca85-5f69-40a9-b45a-3d99121ec395.png)
### 快速实践

#### 项目准备

准备一个Asp.Net Core 6.0的Web(前端简单展示)。按照如上几个用例挨个实现(退出用例不考虑)

![141109567_3dd05954-2d2b-4f03-9479-7c2023870133](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141109567_3dd05954-2d2b-4f03-9479-7c2023870133.png)
#### 增加Account控制器

简单实现，跳过账号验证之类的过程，主要是生成一个Cookie，方便请求其他需要授权验证的接口

```plain
public class AccountController : Controller
{
    public async Task<IActionResult> GenerateCookie(string userName = "9527")
    {
        var claims = new List<Claim>()
        {
            new Claim("id", "1"),
            new Claim(ClaimTypes.Name,userName),
            new Claim(ClaimTypes.Role,"Admin"),
        };
        var claimsIdentity = new ClaimsIdentity(claims, "Custom");
        var claimsPrincipal = new ClaimsPrincipal(claimsIdentity);


        await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, claimsPrincipal, new AuthenticationProperties
        {
            ExpiresUtc = DateTime.Now.AddMinutes(30)
        });


        return Content(HttpContext.Response.Headers.SetCookie);
    }
}
```
#### 服务注册

```plain
using Microsoft.AspNetCore.Authentication.Cookies;


var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();//包含Authorize需要的服务
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie();//使用Cookie作为鉴权Schema


var app = builder.Build();
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();//使用鉴权中间件
app.UseAuthorization();
app.MapDefaultControllerRoute();
app.Run();
```
#### 获取Cookie

简单点，直接Postman请求拿到Cookie，然后便方便对其他接口测试。

![141110683_abeb0385-4bec-46ca-90d5-37c9faab46c1](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141110683_abeb0385-4bec-46ca-90d5-37c9faab46c1.png)
#### 资源控制

有些时候需要验证下Authorize中Policy的一些要求，或是做一些自定义扩展，得设置Authorize特性，全局设置，Controller上设置或是Action都行。

```plain
public class HomeController : Controller
{
    public IActionResult Index()
    {
        return View();
    }


    [Authorize(Roles = "Admin")]
    public IActionResult Auth()
    {
        var names = HttpContext.User.FindAll(c => c.Type == ClaimTypes.Name);
        ViewBag.CurrentUserName = names != null && names.Any() ? names.First().Value : "tester";
        return View();
    }
}
```
#### 访问资源

![141112485_7550bf1b-31dc-4928-b03b-acd6dd8972c8](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141112485_7550bf1b-31dc-4928-b03b-acd6dd8972c8.png)
如此一来，当想要尝试些扩展，比如自定义IClaimTransformation，AuthorizationHandler，AuthorizationPolicyProvider之类时候，想要快速搭建Demo时，便少了些临时去搜资料找方案的过程了。


>2023-01-15,望技术有成后能回来看见自己的脚步

