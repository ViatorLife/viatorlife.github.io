[https://www.cnblogs.com/CKExp/p/17157754.html](https://www.cnblogs.com/CKExp/p/17157754.html)


## 前言

有时候想快速验证一些想法，新建一个控制台来弄，可控制台模板是轻量级的应用程序模板，不具备配置、日志、依赖注入等一些功能。


## Configuration

在Asp.Net Core应用程序中，可以通过依赖注入使用IConfiguration接口来使用配置。而控制台模板十分简单，没有内置依赖注入，应用程序所依赖的功能(如配置)不容易获得。因此需要一点点在控制台中搭建最为基本的功能，想要使用配置，可通过ConfigurationBuilder类来创建Configuration，还可以使用Json文件、机密文件、环境变量、命令行或自定义的配置提供程序来作为配置源获取配置信息。


### 安装Configuration包

安装相关NuGet包，每个包的用途看名字就很一目了然。

```plain
Install-Package Microsoft.Extensions.Configuration
Install-Package Microsoft.Extensions.Configuration.Json
Install-Package Microsoft.Extensions.Configuration.CommandLine
Install-Package Microsoft.Extensions.Configuration.UserSecrets
Install-Package Microsoft.Extensions.Configuration.EnvironmentVariables 
```


### 组装Configuration

如下代码中，添加了一个appsetings.json的配置文件并设置其属性复制到项目输出文件夹，还添加了环境变量、用户机密和命令行作为配置源。最后构建Configuration。

```plain
using Microsoft.Extensions.Configuration;


IConfiguration Configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
    .AddUserSecrets<Program>()
    .AddEnvironmentVariables()
    .AddCommandLine(args)
    .Build();
```
如上配置源会由后者优先级最高覆盖前者相同配置。

### 使用Configuration

已经构建了配置提供程序，可以按照下面的方式使用它们

```plain
// 获取配置节
var section = Configuration.GetSection("ConnectionStrings");


// 获取值
var value = Configuration.GetValue("ConnectionStrings:Default");
```


## 强类型对象

配置节可以映射到强类型对象(选项模式)，安装如下Nuget包，可以使用Bind 方法将配置节映射到强类型对象，又或是使用Get方法(推荐)同样的作用。

```plain
Install-Package Microsoft.Extensions.Configuration.Binder
```
例如，有下面的类表示Jwt配置
```plain
public class JwtOptions
{
    public const string Name = "Jwt";


    public string Audience { get; set; }
    public string Issuer { get; set; }
    public double ExpiresMinutes { get; set; } = 30d;
    public string SymmetricSecurityKeyString { get; set; }
}
```
Json配置文件中加入配置节点
```plain
 {
   "Jwt": {
      "Audience": "http://localhost:5105",
      "Issuer": "http://localhost:5105",
      "ExpiresMinutes": 30,
      "SymmetricSecurityKeyString": "Symmetric Security Key"
    }
  }
```
如此可以通过Bind或Get方法获取JwtOptions对象。
```plain
// Bind方法
var jwtOptions = new JwtOptions();
Configuration.GetSection(JwtOptions.Name).Bind(jwtOptions);


// Get方法
var jwtOptions = Configuration.GetSection(JwtOptions.Name).Get<JwtOptions>();
```


>2023-02-27,望技术有成后能回来看见自己的脚步

